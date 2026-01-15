# 認證與 MCP 系統深度分析報告

本報告深入分析 Gemini CLI 的認證架構和 Model Context Protocol (MCP) 系統實現，涵蓋所有認證方法、OAuth 流程、Token 管理、MCP 客戶端架構及安全機制。

---

## 目錄

1. [認證方法概述](#1-認證方法概述)
2. [Token 管理與刷新機制](#2-token-管理與刷新機制)
3. [認證提供者詳解](#3-認證提供者詳解)
4. [OAuth 認證流程深度解析](#4-oauth-認證流程深度解析)
5. [MCP 客戶端實現](#5-mcp-客戶端實現)
6. [MCP 伺服器配置](#6-mcp-伺服器配置)
7. [傳輸層實現](#7-傳輸層實現)
8. [MCP 工具發現與註冊](#8-mcp-工具發現與註冊)
9. [Token 儲存實現](#9-token-儲存實現)
10. [安全考量](#10-安全考量)
11. [連接與認證流程圖](#11-連接與認證流程圖)

---

## 1. 認證方法概述

### 1.1 五種主要認證類型 (AuthType)

Gemini CLI 支援五種認證方法，定義於 `contentGenerator.ts`：

**檔案路徑:** `/packages/core/src/core/contentGenerator.ts` (Lines 49-55)

```typescript
export enum AuthType {
  LOGIN_WITH_GOOGLE = 'oauth-personal',      // OAuth2 Google 個人登入
  USE_GEMINI = 'gemini-api-key',             // Gemini API Key 認證
  USE_VERTEX_AI = 'vertex-ai',               // Vertex AI 憑證認證
  LEGACY_CLOUD_SHELL = 'cloud-shell',        // Cloud Shell ADC (舊版)
  COMPUTE_ADC = 'compute-default-credentials', // Compute Engine ADC
}
```

#### 認證類型詳細說明

| 認證類型 | 標識符 | 使用場景 | 配置來源 |
|---------|--------|----------|----------|
| **LOGIN_WITH_GOOGLE** | `oauth-personal` | 一般使用者透過瀏覽器 OAuth 登入 | 互動式授權流程 |
| **USE_GEMINI** | `gemini-api-key` | 使用 Gemini API Key 的開發者 | `GEMINI_API_KEY` 環境變數或儲存的 key |
| **USE_VERTEX_AI** | `vertex-ai` | 企業用戶使用 Vertex AI | `GOOGLE_API_KEY` 或 `GOOGLE_CLOUD_PROJECT` |
| **LEGACY_CLOUD_SHELL** | `cloud-shell` | Google Cloud Shell 環境 | 自動偵測環境 |
| **COMPUTE_ADC** | `compute-default-credentials` | GCE/GKE 環境的預設憑證 | Application Default Credentials |

### 1.2 MCP 認證提供者類型 (AuthProviderType)

MCP 伺服器支援三種專用的認證提供者：

**檔案路徑:** `/packages/core/src/config/config.ts` (Lines 247-251)

```typescript
export enum AuthProviderType {
  DYNAMIC_DISCOVERY = 'dynamic_discovery',           // 自動發現 OAuth 配置
  GOOGLE_CREDENTIALS = 'google_credentials',         // Google Application Default Credentials
  SERVICE_ACCOUNT_IMPERSONATION = 'service_account_impersonation', // 服務帳戶模擬
}
```

#### AuthProviderType 應用場景對照表

| 提供者類型 | 適用場景 | 必要配置 | Token 類型 |
|-----------|----------|----------|-----------|
| `DYNAMIC_DISCOVERY` | 支援 OAuth 標準發現的 MCP 伺服器 | 僅需 `url` | Access Token |
| `GOOGLE_CREDENTIALS` | Google API 端點 (googleapis.com) | `url`, `oauth.scopes` | Google Access Token |
| `SERVICE_ACCOUNT_IMPERSONATION` | 需要模擬服務帳戶的場景 | `targetServiceAccount`, `targetAudience` | OIDC ID Token |

### 1.3 OAuth 配置介面

MCP OAuth 配置的完整介面定義：

**檔案路徑:** `/packages/core/src/mcp/oauth-provider.ts` (Lines 25-36)

```typescript
export interface MCPOAuthConfig {
  enabled?: boolean;           // 是否啟用 OAuth
  clientId?: string;           // OAuth Client ID
  clientSecret?: string;       // OAuth Client Secret (可選，公開客戶端不需要)
  authorizationUrl?: string;   // 授權端點 URL
  tokenUrl?: string;           // Token 端點 URL
  scopes?: string[];           // 請求的權限範圍
  audiences?: string[];        // Token 目標受眾
  redirectUri?: string;        // 自訂重定向 URI
  tokenParamName?: string;     // SSE 連接時的 token 參數名稱
  registrationUrl?: string;    // 動態客戶端註冊端點
}
```

---

## 2. Token 管理與刷新機制

### 2.1 Token 資料結構

系統定義了兩個核心的 Token 資料結構：

**檔案路徑:** `/packages/core/src/mcp/token-storage/types.ts` (Lines 10-28)

```typescript
/**
 * OAuth Token 結構
 * 包含存取權杖及其相關元資料
 */
export interface OAuthToken {
  accessToken: string;        // 存取權杖
  refreshToken?: string;      // 刷新權杖 (可選)
  expiresAt?: number;         // 過期時間戳 (毫秒)，包含緩衝時間
  tokenType: string;          // Token 類型 (通常為 'Bearer')
  scope?: string;             // 授權範圍
}

/**
 * OAuth 憑證完整結構
 * 包含 Token 及相關的伺服器資訊
 */
export interface OAuthCredentials {
  serverName: string;         // MCP 伺服器名稱
  token: OAuthToken;          // OAuth Token
  clientId?: string;          // 使用的 Client ID
  tokenUrl?: string;          // Token 端點 URL (用於刷新)
  mcpServerUrl?: string;      // MCP 伺服器 URL (用於 resource 參數)
  updatedAt: number;          // 最後更新時間戳
}
```

### 2.2 Token 過期緩衝機制

系統採用 5 分鐘的預緩衝時間，確保 Token 在實際過期前被刷新：

**檔案路徑:** `/packages/core/src/mcp/oauth-utils.ts` (Line 51)

```typescript
export const FIVE_MIN_BUFFER_MS = 5 * 60 * 1000; // 5 分鐘 = 300,000 毫秒
```

#### Token 過期檢查邏輯

```typescript
// 檔案: /packages/core/src/mcp/oauth-token-storage.ts (Lines 204-212)
isTokenExpired(token: OAuthToken): boolean {
  if (!token.expiresAt) {
    return false; // 無過期時間，視為有效
  }

  // 加上 5 分鐘緩衝，提前認定為過期
  const bufferMs = 5 * 60 * 1000;
  return Date.now() + bufferMs >= token.expiresAt;
}
```

### 2.3 Token 刷新實現

Token 刷新由 `MCPOAuthProvider.refreshAccessToken()` 處理：

**檔案路徑:** `/packages/core/src/mcp/oauth-provider.ts` (Lines 587-698)

```typescript
async refreshAccessToken(
  config: MCPOAuthConfig,
  refreshToken: string,
  tokenUrl: string,
  mcpServerUrl?: string,
): Promise<OAuthTokenResponse> {
  // 1. 構建刷新請求參數
  const params = new URLSearchParams({
    grant_type: 'refresh_token',
    refresh_token: refreshToken,
    client_id: config.clientId!,
  });

  // 2. 添加可選參數
  if (config.clientSecret) {
    params.append('client_secret', config.clientSecret);
  }
  if (config.scopes?.length) {
    params.append('scope', config.scopes.join(' '));
  }
  if (config.audiences?.length) {
    params.append('audience', config.audiences.join(' '));
  }

  // 3. RFC 9728: 添加 resource 參數
  if (mcpServerUrl) {
    params.append('resource', OAuthUtils.buildResourceParameter(mcpServerUrl));
  }

  // 4. 發送刷新請求
  const response = await fetch(tokenUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
      Accept: 'application/json, application/x-www-form-urlencoded',
    },
    body: params.toString(),
  });

  // 5. 處理回應 (支援 JSON 和 form-urlencoded 格式)
  // ...
}
```

### 2.4 自動 Token 刷新流程

`getValidToken()` 方法實現了完整的 Token 有效性檢查與自動刷新：

**檔案路徑:** `/packages/core/src/mcp/oauth-provider.ts` (Lines 960-1030)

```typescript
async getValidToken(
  serverName: string,
  config: MCPOAuthConfig,
): Promise<string | null> {
  // 1. 從儲存載入憑證
  const credentials = await this.tokenStorage.getCredentials(serverName);
  if (!credentials) {
    return null; // 無憑證，需要重新認證
  }

  const { token } = credentials;

  // 2. 檢查 Token 是否過期
  if (!this.tokenStorage.isTokenExpired(token)) {
    return token.accessToken; // Token 有效，直接返回
  }

  // 3. 嘗試刷新 Token
  if (token.refreshToken && config.clientId && credentials.tokenUrl) {
    try {
      const newTokenResponse = await this.refreshAccessToken(
        config,
        token.refreshToken,
        credentials.tokenUrl,
        credentials.mcpServerUrl,
      );

      // 4. 更新儲存的 Token
      const newToken: OAuthToken = {
        accessToken: newTokenResponse.access_token,
        tokenType: newTokenResponse.token_type,
        refreshToken: newTokenResponse.refresh_token || token.refreshToken,
        scope: newTokenResponse.scope || token.scope,
      };

      if (newTokenResponse.expires_in) {
        newToken.expiresAt = Date.now() + newTokenResponse.expires_in * 1000;
      }

      await this.tokenStorage.saveToken(
        serverName, newToken, config.clientId,
        credentials.tokenUrl, credentials.mcpServerUrl
      );

      return newToken.accessToken;
    } catch (error) {
      // 5. 刷新失敗，移除無效 Token
      await this.tokenStorage.deleteCredentials(serverName);
    }
  }

  return null; // 無法刷新，需要重新認證
}
```

#### Token 刷新流程圖

```
                            ┌─────────────────────┐
                            │  getValidToken()    │
                            └──────────┬──────────┘
                                       │
                            ┌──────────▼──────────┐
                            │  載入儲存的憑證      │
                            └──────────┬──────────┘
                                       │
                   ┌───────────────────┴───────────────────┐
                   │                                       │
          找到憑證 │                                       │ 無憑證
                   ▼                                       ▼
       ┌───────────────────────┐               ┌──────────────────┐
       │  檢查 Token 是否過期   │               │  返回 null       │
       │  (含 5 分鐘緩衝)       │               │  (需重新認證)     │
       └───────────┬───────────┘               └──────────────────┘
                   │
          ┌────────┴────────┐
          │                 │
    未過期 │                 │ 已過期
          ▼                 ▼
┌──────────────────┐  ┌──────────────────────────┐
│  返回 accessToken │  │  有 refreshToken?         │
└──────────────────┘  └───────────┬──────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
               有   │                           │ 無
                    ▼                           ▼
        ┌───────────────────────┐   ┌──────────────────┐
        │  refreshAccessToken() │   │  刪除憑證         │
        └───────────┬───────────┘   │  返回 null       │
                    │               └──────────────────┘
          ┌─────────┴─────────┐
          │                   │
     成功 │                   │ 失敗
          ▼                   ▼
 ┌──────────────────┐  ┌──────────────────┐
 │  儲存新 Token     │  │  刪除無效憑證     │
 │  返回 accessToken │  │  返回 null       │
 └──────────────────┘  └──────────────────┘
```

---

## 3. 認證提供者詳解

### 3.1 Google Credentials 提供者

此提供者使用 Google Application Default Credentials (ADC) 獲取 Token：

**檔案路徑:** `/packages/core/src/mcp/google-auth-provider.ts` (Lines 21-157)

```typescript
export class GoogleCredentialProvider implements McpAuthProvider {
  private readonly auth: GoogleAuth;
  private cachedToken?: OAuthTokens;
  private tokenExpiryTime?: number;

  // 允許的主機白名單
  private static readonly ALLOWED_HOSTS = [
    /^.+\.googleapis\.com$/,    // Google API 端點
    /^(.*\.)?luci\.app$/        // LUCI 應用
  ];

  constructor(private readonly config?: MCPServerConfig) {
    const url = this.config?.url || this.config?.httpUrl;

    // 1. 驗證 URL 必須存在
    if (!url) {
      throw new Error('URL must be provided for Google Credentials provider');
    }

    // 2. 驗證主機在白名單中
    const hostname = new URL(url).hostname;
    if (!ALLOWED_HOSTS.some((pattern) => pattern.test(hostname))) {
      throw new Error(`Host "${hostname}" is not an allowed host`);
    }

    // 3. 驗證必須提供 scopes
    const scopes = this.config?.oauth?.scopes;
    if (!scopes || scopes.length === 0) {
      throw new Error('Scopes must be provided for Google Credentials provider');
    }

    // 4. 初始化 GoogleAuth
    this.auth = new GoogleAuth({ scopes });
  }

  async tokens(): Promise<OAuthTokens | undefined> {
    // 檢查快取 Token 是否有效
    if (this.cachedToken && this.tokenExpiryTime &&
        Date.now() < this.tokenExpiryTime - FIVE_MIN_BUFFER_MS) {
      return this.cachedToken;
    }

    // 從 ADC 獲取新 Token
    const client = await this.auth.getClient();
    const accessTokenResponse = await client.getAccessToken();

    if (!accessTokenResponse.token) {
      return undefined;
    }

    // 快取新 Token
    const newToken: OAuthTokens = {
      access_token: accessTokenResponse.token,
      token_type: 'Bearer',
    };

    const expiryTime = client.credentials?.expiry_date;
    if (expiryTime) {
      this.tokenExpiryTime = expiryTime;
      this.cachedToken = newToken;
    }

    return newToken;
  }

  // 返回配額專案 ID 的 header
  async getRequestHeaders(): Promise<Record<string, string>> {
    const headers: Record<string, string> = {};
    const quotaProjectId = await this.getQuotaProjectId();
    if (quotaProjectId) {
      headers['X-Goog-User-Project'] = quotaProjectId;
    }
    return headers;
  }
}
```

#### Google Credentials 提供者配置範例

```json
{
  "mcpServers": {
    "google-drive-mcp": {
      "url": "https://mcp.googleapis.com/drive/v1",
      "authProviderType": "google_credentials",
      "oauth": {
        "scopes": ["https://www.googleapis.com/auth/drive.readonly"]
      }
    }
  }
}
```

### 3.2 服務帳戶模擬提供者

此提供者透過 IAM Credentials API 模擬目標服務帳戶，生成 ID Token：

**檔案路徑:** `/packages/core/src/mcp/sa-impersonation-provider.ts` (Lines 25-156)

```typescript
export class ServiceAccountImpersonationProvider implements McpAuthProvider {
  private readonly targetServiceAccount: string;  // 目標服務帳戶
  private readonly targetAudience: string;        // OAuth Client ID
  private readonly auth: GoogleAuth;
  private cachedToken?: OAuthTokens;
  private tokenExpiryTime?: number;

  constructor(private readonly config: MCPServerConfig) {
    // 1. 驗證必須提供 URL
    if (!this.config.httpUrl && !this.config.url) {
      throw new Error('A url or httpUrl must be provided');
    }

    // 2. 驗證必須提供 targetAudience
    if (!config.targetAudience) {
      throw new Error('targetAudience must be provided');
    }
    this.targetAudience = config.targetAudience;

    // 3. 驗證必須提供 targetServiceAccount
    if (!config.targetServiceAccount) {
      throw new Error('targetServiceAccount must be provided');
    }
    this.targetServiceAccount = config.targetServiceAccount;

    this.auth = new GoogleAuth();
  }

  async tokens(): Promise<OAuthTokens | undefined> {
    // 1. 檢查快取 Token
    if (this.cachedToken && this.tokenExpiryTime &&
        Date.now() < this.tokenExpiryTime - FIVE_MIN_BUFFER_MS) {
      return this.cachedToken;
    }

    // 2. 清除過期快取
    this.cachedToken = undefined;
    this.tokenExpiryTime = undefined;

    // 3. 呼叫 IAM Credentials API 生成 ID Token
    const client = await this.auth.getClient();
    const url = `https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/${
      encodeURIComponent(this.targetServiceAccount)
    }:generateIdToken`;

    const res = await client.request<{ token: string }>({
      url,
      method: 'POST',
      data: {
        audience: this.targetAudience,
        includeEmail: true,
      },
    });

    const idToken = res.data.token;

    // 4. 解析 JWT 過期時間
    const expiryTime = OAuthUtils.parseTokenExpiry(idToken);

    // 5. 將 ID Token 作為 access_token 使用
    const newTokens: OAuthTokens = {
      access_token: idToken,
      token_type: 'Bearer',
    };

    if (expiryTime) {
      this.tokenExpiryTime = expiryTime;
      this.cachedToken = newTokens;
    }

    return newTokens;
  }
}
```

#### JWT 過期時間解析

**檔案路徑:** `/packages/core/src/mcp/oauth-utils.ts` (Lines 411-429)

```typescript
static parseTokenExpiry(idToken: string): number | undefined {
  try {
    // JWT 格式: header.payload.signature
    const payload = JSON.parse(
      Buffer.from(idToken.split('.')[1], 'base64').toString()
    );

    if (payload && typeof payload.exp === 'number') {
      // JWT 'exp' 是秒，轉換為毫秒
      return payload.exp * 1000;
    }
  } catch (e) {
    debugLogger.error('Failed to parse ID token for expiry time:', e);
  }
  return undefined;
}
```

#### 服務帳戶模擬配置範例

```json
{
  "mcpServers": {
    "internal-mcp-server": {
      "url": "https://mcp.internal.company.com/api",
      "authProviderType": "service_account_impersonation",
      "targetServiceAccount": "mcp-client@project-123.iam.gserviceaccount.com",
      "targetAudience": "123456789.apps.googleusercontent.com"
    }
  }
}
```

### 3.3 McpAuthProvider 介面

所有 MCP 認證提供者必須實作此介面：

**檔案路徑:** `/packages/core/src/mcp/auth-provider.ts` (Lines 7-18)

```typescript
import type { OAuthClientProvider } from '@modelcontextprotocol/sdk/client/auth.js';

/**
 * 擴展 OAuthClientProvider，允許提供者注入自訂 headers
 */
export interface McpAuthProvider extends OAuthClientProvider {
  /**
   * 返回要添加到請求的自訂 headers
   */
  getRequestHeaders?(): Promise<Record<string, string>>;
}
```

---

## 4. OAuth 認證流程深度解析

### 4.1 PKCE (Proof Key for Code Exchange) 實現

PKCE 是 OAuth 2.0 的安全擴展 (RFC 7636)，用於防止授權碼攔截攻擊：

**檔案路徑:** `/packages/core/src/mcp/oauth-provider.ts` (Lines 84-91, 246-260)

```typescript
/**
 * PKCE 參數結構
 */
interface PKCEParams {
  codeVerifier: string;   // 原始驗證碼 (43-128 字元)
  codeChallenge: string;  // SHA256 雜湊後的挑戰碼
  state: string;          // CSRF 保護用的隨機狀態值
}

/**
 * 生成 PKCE 參數
 */
private generatePKCEParams(): PKCEParams {
  // 1. 生成 code_verifier (32 位元組 = 43 字元 base64url)
  const codeVerifier = crypto.randomBytes(32).toString('base64url');

  // 2. 計算 code_challenge = BASE64URL(SHA256(code_verifier))
  const codeChallenge = crypto
    .createHash('sha256')
    .update(codeVerifier)
    .digest('base64url');

  // 3. 生成 state 參數 (16 位元組隨機值)
  const state = crypto.randomBytes(16).toString('base64url');

  return { codeVerifier, codeChallenge, state };
}
```

#### PKCE 流程圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PKCE 授權流程                                 │
└─────────────────────────────────────────────────────────────────────┘

    Gemini CLI                Authorization Server               Browser
        │                             │                              │
        │  1. 生成 PKCE 參數           │                              │
        │  ┌────────────────────┐     │                              │
        │  │ code_verifier:     │     │                              │
        │  │   random(32 bytes) │     │                              │
        │  │                    │     │                              │
        │  │ code_challenge:    │     │                              │
        │  │   SHA256(verifier) │     │                              │
        │  │   .base64url()     │     │                              │
        │  │                    │     │                              │
        │  │ state:             │     │                              │
        │  │   random(16 bytes) │     │                              │
        │  └────────────────────┘     │                              │
        │                             │                              │
        │  2. 啟動 Callback Server    │                              │
        │  (port 0 = OS 分配)         │                              │
        │                             │                              │
        │  3. 建構授權 URL             │                              │
        │─────────────────────────────┼─────────────────────────────▶│
        │  GET /authorize?            │                              │
        │    client_id=xxx            │                              │
        │    response_type=code       │                              │
        │    redirect_uri=localhost   │                              │
        │    code_challenge=xxx       │                              │
        │    code_challenge_method=S256                              │
        │    state=xxx                │                              │
        │    scope=xxx                │                              │
        │    resource=mcp_server_url  │                              │
        │                             │                              │
        │                             │  4. 使用者授權                 │
        │                             │◀─────────────────────────────│
        │                             │                              │
        │  5. 重定向回 Callback        │                              │
        │◀────────────────────────────│                              │
        │  ?code=xxx&state=xxx        │                              │
        │                             │                              │
        │  6. 驗證 state              │                              │
        │  (防止 CSRF)                 │                              │
        │                             │                              │
        │  7. 交換 Token              │                              │
        │────────────────────────────▶│                              │
        │  POST /token                │                              │
        │    grant_type=authorization_code                           │
        │    code=xxx                 │                              │
        │    redirect_uri=localhost   │                              │
        │    code_verifier=xxx        │  ← 原始 verifier，驗證         │
        │    client_id=xxx            │    SHA256(verifier)==challenge│
        │                             │                              │
        │  8. 返回 Token              │                              │
        │◀────────────────────────────│                              │
        │  { access_token, refresh_token, expires_in }               │
        │                             │                              │
```

### 4.2 動態客戶端註冊 (RFC 7591)

當沒有預先配置的 `clientId` 時，系統會嘗試動態客戶端註冊：

**檔案路徑:** `/packages/core/src/mcp/oauth-provider.ts` (Lines 57-82, 114-147)

```typescript
/**
 * 動態客戶端註冊請求 (RFC 7591)
 */
export interface OAuthClientRegistrationRequest {
  client_name: string;                    // 客戶端名稱
  redirect_uris: string[];                // 重定向 URI 列表
  grant_types: string[];                  // 授權類型
  response_types: string[];               // 回應類型
  token_endpoint_auth_method: string;     // Token 端點認證方法
  scope?: string;                         // 請求的範圍
}

/**
 * 動態客戶端註冊回應
 */
export interface OAuthClientRegistrationResponse {
  client_id: string;                      // 分配的 Client ID
  client_secret?: string;                 // Client Secret (可選)
  client_id_issued_at?: number;           // Client ID 發放時間
  client_secret_expires_at?: number;      // Client Secret 過期時間
  redirect_uris: string[];
  grant_types: string[];
  response_types: string[];
  token_endpoint_auth_method: string;
  scope?: string;
}

/**
 * 執行動態客戶端註冊
 */
private async registerClient(
  registrationUrl: string,
  config: MCPOAuthConfig,
  redirectPort: number,
): Promise<OAuthClientRegistrationResponse> {
  const redirectUri = config.redirectUri ||
    `http://localhost:${redirectPort}/oauth/callback`;

  const registrationRequest: OAuthClientRegistrationRequest = {
    client_name: 'Gemini CLI MCP Client',
    redirect_uris: [redirectUri],
    grant_types: ['authorization_code', 'refresh_token'],
    response_types: ['code'],
    token_endpoint_auth_method: 'none', // 公開客戶端
    scope: config.scopes?.join(' ') || '',
  };

  const response = await fetch(registrationUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(registrationRequest),
  });

  if (!response.ok) {
    throw new Error(`Client registration failed: ${response.status}`);
  }

  return await response.json();
}
```

### 4.3 RFC 9728 受保護資源元資料

系統實現了 RFC 9728 (OAuth 2.0 Protected Resource Metadata) 標準：

**檔案路徑:** `/packages/core/src/mcp/oauth-utils.ts` (Lines 38-49, 233-312)

```typescript
/**
 * OAuth 受保護資源元資料 (RFC 9728)
 */
export interface OAuthProtectedResourceMetadata {
  resource: string;                              // 資源識別符
  authorization_servers?: string[];              // 授權伺服器列表
  bearer_methods_supported?: string[];           // 支援的 Bearer 方法
  resource_documentation?: string;               // 資源文件 URL
  resource_signing_alg_values_supported?: string[];
  resource_encryption_alg_values_supported?: string[];
  resource_encryption_enc_values_supported?: string[];
}

/**
 * 從 MCP 伺服器發現 OAuth 配置
 */
static async discoverOAuthConfig(
  serverUrl: string,
): Promise<MCPOAuthConfig | null> {
  // 1. 嘗試 root-based 發現
  const wellKnownUrls = this.buildWellKnownUrls(serverUrl, false);
  let resourceMetadata = await this.fetchProtectedResourceMetadata(
    wellKnownUrls.protectedResource
  );

  // 2. 如果失敗，嘗試 path-based 發現
  if (!resourceMetadata) {
    const url = new URL(serverUrl);
    if (url.pathname && url.pathname !== '/') {
      const pathBasedUrls = this.buildWellKnownUrls(serverUrl, true);
      resourceMetadata = await this.fetchProtectedResourceMetadata(
        pathBasedUrls.protectedResource
      );
    }
  }

  // 3. RFC 9728 Section 7.3: 驗證 resource 參數匹配
  if (resourceMetadata) {
    const expectedResource = this.buildResourceParameter(serverUrl);
    if (resourceMetadata.resource !== expectedResource) {
      throw new ResourceMismatchError(
        `Protected resource ${resourceMetadata.resource} does not match ${expectedResource}`
      );
    }
  }

  // 4. 從授權伺服器獲取元資料
  if (resourceMetadata?.authorization_servers?.length) {
    const authServerUrl = resourceMetadata.authorization_servers[0];
    const authServerMetadata = await this.discoverAuthorizationServerMetadata(authServerUrl);

    if (authServerMetadata) {
      return this.metadataToOAuthConfig(authServerMetadata);
    }
  }

  return null;
}
```

#### Well-Known URL 建構

```typescript
/**
 * 建構 well-known OAuth 端點 URL
 */
static buildWellKnownUrls(baseUrl: string, includePathSuffix = false) {
  const serverUrl = new URL(baseUrl);
  const base = `${serverUrl.protocol}//${serverUrl.host}`;

  if (!includePathSuffix) {
    // 標準 root-based 發現
    return {
      protectedResource: `${base}/.well-known/oauth-protected-resource`,
      authorizationServer: `${base}/.well-known/oauth-authorization-server`,
    };
  }

  // Path-based 發現 (用於如 Keycloak 的路徑型授權伺服器)
  const pathSuffix = serverUrl.pathname.replace(/\/$/, '');
  return {
    protectedResource: `${base}/.well-known/oauth-protected-resource${pathSuffix}`,
    authorizationServer: `${base}/.well-known/oauth-authorization-server${pathSuffix}`,
  };
}
```

### 4.4 授權伺服器元資料發現 (RFC 8414)

**檔案路徑:** `/packages/core/src/mcp/oauth-utils.ts` (Lines 22-36, 164-225)

```typescript
/**
 * OAuth 授權伺服器元資料 (RFC 8414)
 */
export interface OAuthAuthorizationServerMetadata {
  issuer: string;                           // 發行者識別符
  authorization_endpoint: string;           // 授權端點
  token_endpoint: string;                   // Token 端點
  token_endpoint_auth_methods_supported?: string[];
  revocation_endpoint?: string;             // 撤銷端點
  registration_endpoint?: string;           // 客戶端註冊端點
  response_types_supported?: string[];
  grant_types_supported?: string[];
  code_challenge_methods_supported?: string[];  // PKCE 支援
  scopes_supported?: string[];              // 支援的範圍
}

/**
 * 發現授權伺服器元資料
 * 嘗試多種 well-known 端點
 */
static async discoverAuthorizationServerMetadata(
  authServerUrl: string,
): Promise<OAuthAuthorizationServerMetadata | null> {
  const authServerUrlObj = new URL(authServerUrl);
  const base = `${authServerUrlObj.protocol}//${authServerUrlObj.host}`;

  const endpointsToTry: string[] = [];

  // 對於有路徑的 issuer URL，按順序嘗試：
  if (authServerUrlObj.pathname !== '/') {
    // 1. OAuth 2.0 元資料 (path insertion)
    endpointsToTry.push(
      `${base}/.well-known/oauth-authorization-server${authServerUrlObj.pathname}`
    );

    // 2. OIDC Discovery (path insertion)
    endpointsToTry.push(
      `${base}/.well-known/openid-configuration${authServerUrlObj.pathname}`
    );

    // 3. OIDC Discovery (path appending)
    endpointsToTry.push(
      `${base}${authServerUrlObj.pathname}/.well-known/openid-configuration`
    );
  }

  // 4. 標準 OAuth 2.0 元資料
  endpointsToTry.push(`${base}/.well-known/oauth-authorization-server`);

  // 5. 標準 OIDC Discovery
  endpointsToTry.push(`${base}/.well-known/openid-configuration`);

  // 依序嘗試每個端點
  for (const endpoint of endpointsToTry) {
    const metadata = await this.fetchAuthorizationServerMetadata(endpoint);
    if (metadata) return metadata;
  }

  return null;
}
```

### 4.5 完整認證流程

**檔案路徑:** `/packages/core/src/mcp/oauth-provider.ts` (Lines 709-951)

```typescript
async authenticate(
  serverName: string,
  config: MCPOAuthConfig,
  mcpServerUrl?: string,
  events?: EventEmitter,
): Promise<OAuthToken> {
  // === 階段 1: OAuth 發現 ===
  if (!config.authorizationUrl && mcpServerUrl) {
    // 1a. 檢查 WWW-Authenticate header
    try {
      const response = await fetch(mcpServerUrl, {
        method: 'HEAD',
        headers: { Accept: OAuthUtils.isSSEEndpoint(mcpServerUrl)
          ? 'text/event-stream' : 'application/json' },
      });

      if (response.status === 401 || response.status === 307) {
        const wwwAuthenticate = response.headers.get('www-authenticate');
        if (wwwAuthenticate) {
          const discoveredConfig = await OAuthUtils.discoverOAuthFromWWWAuthenticate(
            wwwAuthenticate, mcpServerUrl
          );
          if (discoveredConfig) {
            config = { ...config, ...discoveredConfig };
          }
        }
      }
    } catch (error) {
      // 處理發現錯誤
    }

    // 1b. 標準 well-known 發現
    if (!config.authorizationUrl) {
      const discoveredConfig = await this.discoverOAuthFromMCPServer(mcpServerUrl);
      if (discoveredConfig) {
        config = { ...config, ...discoveredConfig };
      }
    }
  }

  // === 階段 2: PKCE 生成 ===
  const pkceParams = this.generatePKCEParams();

  // === 階段 3: 啟動 Callback 伺服器 ===
  const callbackServer = this.startCallbackServer(pkceParams.state);
  const redirectPort = await callbackServer.port;

  // === 階段 4: 動態客戶端註冊 (如果需要) ===
  if (!config.clientId) {
    let registrationUrl = config.registrationUrl;
    if (!registrationUrl) {
      const { metadata } = await this.discoverAuthServerMetadataForRegistration(
        config.authorizationUrl!
      );
      registrationUrl = metadata.registration_endpoint;
    }

    if (registrationUrl) {
      const registration = await this.registerClient(
        registrationUrl, config, redirectPort
      );
      config.clientId = registration.client_id;
      config.clientSecret = registration.client_secret;
    }
  }

  // === 階段 5: 建構授權 URL ===
  const authUrl = this.buildAuthorizationUrl(
    config, pkceParams, redirectPort, mcpServerUrl
  );

  // === 階段 6: 開啟瀏覽器 ===
  displayMessage(`→ Opening your browser for OAuth sign-in...`);
  await openBrowserSecurely(authUrl);

  // === 階段 7: 等待 Callback ===
  const { code } = await callbackServer.response;

  // === 階段 8: 交換 Token ===
  const tokenResponse = await this.exchangeCodeForToken(
    config, code, pkceParams.codeVerifier, redirectPort, mcpServerUrl
  );

  // === 階段 9: 儲存 Token ===
  const token: OAuthToken = {
    accessToken: tokenResponse.access_token,
    tokenType: tokenResponse.token_type || 'Bearer',
    refreshToken: tokenResponse.refresh_token,
    scope: tokenResponse.scope,
    expiresAt: tokenResponse.expires_in
      ? Date.now() + tokenResponse.expires_in * 1000
      : undefined,
  };

  await this.tokenStorage.saveToken(
    serverName, token, config.clientId, config.tokenUrl, mcpServerUrl
  );

  // === 階段 10: 驗證儲存 ===
  const savedToken = await this.tokenStorage.getCredentials(serverName);
  if (savedToken?.token?.accessToken) {
    const fingerprint = crypto.createHash('sha256')
      .update(savedToken.token.accessToken)
      .digest('hex').slice(0, 8);
    debugLogger.debug(`Token verification successful (fingerprint: ${fingerprint})`);
  }

  return token;
}
```

---

## 5. MCP 客戶端實現

### 5.1 McpClient 類別架構

`McpClient` 負責管理與單一 MCP 伺服器的連接：

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 107-452)

```typescript
export class McpClient {
  private client: Client | undefined;           // MCP SDK 客戶端
  private transport: Transport | undefined;     // 傳輸層
  private status: MCPServerStatus = MCPServerStatus.DISCONNECTED;

  // 工具刷新狀態 (Coalescing Pattern)
  private isRefreshingTools: boolean = false;
  private pendingToolRefresh: boolean = false;
  private isRefreshingResources: boolean = false;
  private pendingResourceRefresh: boolean = false;

  constructor(
    private readonly serverName: string,
    private readonly serverConfig: MCPServerConfig,
    private readonly toolRegistry: ToolRegistry,
    private readonly promptRegistry: PromptRegistry,
    private readonly resourceRegistry: ResourceRegistry,
    private readonly workspaceContext: WorkspaceContext,
    private readonly cliConfig: Config,
    private readonly debugMode: boolean,
    private readonly onToolsUpdated?: (signal?: AbortSignal) => Promise<void>,
  ) {}

  /**
   * 連接到 MCP 伺服器
   */
  async connect(): Promise<void> {
    if (this.status !== MCPServerStatus.DISCONNECTED) {
      throw new Error(`Cannot connect: current state is ${this.status}`);
    }

    this.updateStatus(MCPServerStatus.CONNECTING);
    try {
      this.client = await connectToMcpServer(
        this.serverName,
        this.serverConfig,
        this.debugMode,
        this.workspaceContext,
        this.cliConfig.sanitizationConfig,
      );

      this.registerNotificationHandlers();

      // 錯誤處理
      this.client.onerror = (error) => {
        if (this.status === MCPServerStatus.CONNECTED) {
          coreEvents.emitFeedback('error', `MCP ERROR (${this.serverName})`, error);
          this.updateStatus(MCPServerStatus.DISCONNECTED);
        }
      };

      this.updateStatus(MCPServerStatus.CONNECTED);
    } catch (error) {
      this.updateStatus(MCPServerStatus.DISCONNECTED);
      throw error;
    }
  }

  /**
   * 發現工具和提示
   */
  async discover(cliConfig: Config): Promise<void> {
    this.assertConnected();

    const prompts = await this.discoverPrompts();
    const tools = await this.discoverTools(cliConfig);
    const resources = await this.discoverResources();

    if (prompts.length === 0 && tools.length === 0 && resources.length === 0) {
      throw new Error('No prompts, tools, or resources found');
    }

    for (const tool of tools) {
      this.toolRegistry.registerTool(tool);
    }
    this.toolRegistry.sortTools();
  }

  /**
   * 斷開連接
   */
  async disconnect(): Promise<void> {
    if (this.status !== MCPServerStatus.CONNECTED) return;

    this.toolRegistry.removeMcpToolsByServer(this.serverName);
    this.promptRegistry.removePromptsByServer(this.serverName);
    this.resourceRegistry.removeResourcesByServer(this.serverName);

    this.updateStatus(MCPServerStatus.DISCONNECTING);

    if (this.transport) await this.transport.close();
    if (this.client) await this.client.close();

    this.client = undefined;
    this.updateStatus(MCPServerStatus.DISCONNECTED);
  }
}
```

### 5.2 MCP 伺服器狀態

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 78-99)

```typescript
/**
 * MCP 伺服器連接狀態
 */
export enum MCPServerStatus {
  DISCONNECTED = 'disconnected',     // 已斷開或發生錯誤
  DISCONNECTING = 'disconnecting',   // 正在斷開連接
  CONNECTING = 'connecting',         // 正在連接中
  CONNECTED = 'connected',           // 已連接且可用
}

/**
 * MCP 發現狀態
 */
export enum MCPDiscoveryState {
  NOT_STARTED = 'not_started',       // 尚未開始發現
  IN_PROGRESS = 'in_progress',       // 發現進行中
  COMPLETED = 'completed',           // 發現完成
}
```

### 5.3 McpClientManager 管理器

`McpClientManager` 管理多個 MCP 客戶端的生命週期：

**檔案路徑:** `/packages/core/src/tools/mcp-client-manager.ts` (Lines 28-358)

```typescript
export class McpClientManager {
  private clients: Map<string, McpClient> = new Map();
  private discoveryPromise: Promise<void> | undefined;
  private discoveryState: MCPDiscoveryState = MCPDiscoveryState.NOT_STARTED;

  constructor(
    private readonly toolRegistry: ToolRegistry,
    private readonly cliConfig: Config,
    private readonly eventEmitter?: EventEmitter,
  ) {}

  /**
   * 啟動所有配置的 MCP 伺服器
   */
  async startConfiguredMcpServers(): Promise<void> {
    if (!this.cliConfig.isTrustedFolder()) return;

    const servers = populateMcpServerCommand(
      this.cliConfig.getMcpServers() || {},
      this.cliConfig.getMcpServerCommand(),
    );

    await Promise.all(
      Object.entries(servers).map(([name, config]) =>
        this.maybeDiscoverMcpServer(name, config)
      ),
    );
  }

  /**
   * 啟動擴充功能的 MCP 伺服器
   */
  async startExtension(extension: GeminiCLIExtension): Promise<void> {
    await Promise.all(
      Object.entries(extension.mcpServers ?? {}).map(([name, config]) =>
        this.maybeDiscoverMcpServer(name, { ...config, extension })
      ),
    );
  }

  /**
   * 停止擴充功能的 MCP 伺服器
   */
  async stopExtension(extension: GeminiCLIExtension): Promise<void> {
    await Promise.all(
      Object.keys(extension.mcpServers ?? {}).map(
        this.disconnectClient.bind(this)
      ),
    );
  }

  /**
   * 檢查是否允許連接到特定 MCP 伺服器
   */
  private isAllowedMcpServer(name: string): boolean {
    const allowedNames = this.cliConfig.getAllowedMcpServers();
    if (allowedNames?.length && !allowedNames.includes(name)) {
      return false;
    }

    const blockedNames = this.cliConfig.getBlockedMcpServers();
    if (blockedNames?.length && blockedNames.includes(name)) {
      return false;
    }

    return true;
  }

  /**
   * 重啟所有 MCP 客戶端
   */
  async restart(): Promise<void> {
    await Promise.all(
      Array.from(this.clients.entries()).map(async ([name, client]) => {
        await this.maybeDiscoverMcpServer(name, client.getServerConfig());
      }),
    );
  }

  /**
   * 停止所有 MCP 伺服器
   */
  async stop(): Promise<void> {
    await Promise.all(
      Array.from(this.clients.values()).map((client) => client.disconnect())
    );
    this.clients.clear();
  }
}
```

---

## 6. MCP 伺服器配置

### 6.1 MCPServerConfig 完整結構

**檔案路徑:** `/packages/core/src/config/config.ts` (Lines 208-244)

```typescript
export class MCPServerConfig {
  constructor(
    // === Stdio 傳輸配置 ===
    readonly command?: string,                    // 執行命令
    readonly args?: string[],                     // 命令參數
    readonly env?: Record<string, string>,        // 環境變數
    readonly cwd?: string,                        // 工作目錄

    // === SSE 傳輸配置 ===
    readonly url?: string,                        // SSE/HTTP URL

    // === HTTP Streamable 傳輸配置 ===
    readonly httpUrl?: string,                    // HTTP URL (已棄用)
    readonly headers?: Record<string, string>,    // 自訂 HTTP headers

    // === WebSocket 傳輸配置 ===
    readonly tcp?: string,                        // TCP 連接字串

    // === 傳輸類型指定 ===
    readonly type?: 'sse' | 'http',               // 明確指定傳輸類型

    // === 通用配置 ===
    readonly timeout?: number,                    // 超時時間 (毫秒)
    readonly trust?: boolean,                     // 是否信任此伺服器

    // === 元資料 ===
    readonly description?: string,                // 伺服器描述
    readonly includeTools?: string[],             // 工具白名單
    readonly excludeTools?: string[],             // 工具黑名單
    readonly extension?: GeminiCLIExtension,      // 所屬擴充功能

    // === OAuth 配置 ===
    readonly oauth?: MCPOAuthConfig,              // OAuth 設定
    readonly authProviderType?: AuthProviderType, // 認證提供者類型

    // === 服務帳戶配置 ===
    readonly targetAudience?: string,             // OAuth Client ID
    readonly targetServiceAccount?: string,       // 服務帳戶 email
  ) {}
}
```

### 6.2 配置範例

#### Stdio 傳輸 (本地程序)

```json
{
  "mcpServers": {
    "local-tool": {
      "command": "npx",
      "args": ["-y", "@example/mcp-tool"],
      "env": {
        "API_KEY": "${API_KEY}"
      },
      "cwd": "/path/to/project",
      "timeout": 30000,
      "trust": true
    }
  }
}
```

#### SSE 傳輸 (Server-Sent Events)

```json
{
  "mcpServers": {
    "remote-sse-server": {
      "url": "https://mcp.example.com/sse",
      "type": "sse",
      "headers": {
        "X-Custom-Header": "value"
      },
      "timeout": 60000
    }
  }
}
```

#### HTTP Streamable 傳輸

```json
{
  "mcpServers": {
    "remote-http-server": {
      "url": "https://mcp.example.com/api",
      "type": "http",
      "oauth": {
        "enabled": true,
        "scopes": ["read", "write"]
      }
    }
  }
}
```

#### Google Credentials 認證

```json
{
  "mcpServers": {
    "google-api-server": {
      "url": "https://mcp.googleapis.com/v1",
      "authProviderType": "google_credentials",
      "oauth": {
        "scopes": ["https://www.googleapis.com/auth/cloud-platform"]
      }
    }
  }
}
```

#### 服務帳戶模擬

```json
{
  "mcpServers": {
    "internal-server": {
      "url": "https://internal.company.com/mcp",
      "authProviderType": "service_account_impersonation",
      "targetServiceAccount": "mcp-client@project.iam.gserviceaccount.com",
      "targetAudience": "123456789.apps.googleusercontent.com"
    }
  }
}
```

---

## 7. 傳輸層實現

### 7.1 傳輸類型概覽

| 傳輸類型 | 類別 | 使用場景 | 協議 |
|---------|------|----------|------|
| **Stdio** | `StdioClientTransport` | 本地子程序 MCP 伺服器 | stdin/stdout |
| **SSE** | `SSEClientTransport` | 遠端伺服器 (單向串流) | HTTP + Server-Sent Events |
| **HTTP Streamable** | `StreamableHTTPClientTransport` | 遠端伺服器 (雙向串流) | HTTP + 類 WebSocket |

### 7.2 傳輸選擇邏輯

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 1670-1715)

```typescript
function createUrlTransport(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  transportOptions: StreamableHTTPClientTransportOptions | SSEClientTransportOptions,
): StreamableHTTPClientTransport | SSEClientTransport {

  // 優先順序 1: httpUrl (已棄用，但仍支援)
  if (mcpServerConfig.httpUrl) {
    if (mcpServerConfig.url) {
      debugLogger.warn(
        `Both 'httpUrl' and 'url' configured. Using deprecated 'httpUrl'.`
      );
    }
    return new StreamableHTTPClientTransport(
      new URL(mcpServerConfig.httpUrl), transportOptions
    );
  }

  // 優先順序 2: url + type: 'http'
  if (mcpServerConfig.url && mcpServerConfig.type === 'http') {
    return new StreamableHTTPClientTransport(
      new URL(mcpServerConfig.url), transportOptions
    );
  }

  // 優先順序 3: url + type: 'sse'
  if (mcpServerConfig.url && mcpServerConfig.type === 'sse') {
    return new SSEClientTransport(
      new URL(mcpServerConfig.url), transportOptions
    );
  }

  // 優先順序 4: url 無明確 type (預設 HTTP)
  if (mcpServerConfig.url) {
    return new StreamableHTTPClientTransport(
      new URL(mcpServerConfig.url), transportOptions
    );
  }

  throw new Error(`No URL configured for MCP server '${mcpServerName}'`);
}
```

### 7.3 Stdio 傳輸實現

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 1789-1809)

```typescript
if (mcpServerConfig.command) {
  const transport = new StdioClientTransport({
    command: mcpServerConfig.command,
    args: mcpServerConfig.args || [],
    env: {
      // 淨化父程序環境變數
      ...sanitizeEnvironment(process.env, sanitizationConfig),
      // 覆蓋配置的環境變數
      ...(mcpServerConfig.env || {}),
    } as Record<string, string>,
    cwd: mcpServerConfig.cwd,
    stderr: 'pipe', // 捕獲 stderr 用於調試
  });

  // 調試模式下記錄 stderr
  if (debugMode) {
    transport.stderr!.on('data', (data) => {
      debugLogger.debug(`[MCP STDERR (${mcpServerName})]: ${data.toString().trim()}`);
    });
  }

  return transport;
}
```

### 7.4 OAuth Token 注入

連接前，系統會檢查並注入 OAuth Token：

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 1742-1786)

```typescript
if (mcpServerConfig.httpUrl || mcpServerConfig.url) {
  const authProvider = createAuthProvider(mcpServerConfig);
  const headers: Record<string, string> =
    (await authProvider?.getRequestHeaders?.()) ?? {};

  if (authProvider === undefined) {
    // 檢查 OAuth 配置或儲存的 Token
    let accessToken: string | null = null;

    if (mcpServerConfig.oauth?.enabled) {
      const tokenStorage = new MCPOAuthTokenStorage();
      const mcpAuthProvider = new MCPOAuthProvider(tokenStorage);
      accessToken = await mcpAuthProvider.getValidToken(
        mcpServerName, mcpServerConfig.oauth
      );
    } else {
      // 檢查先前認證的儲存 Token
      accessToken = await getStoredOAuthToken(mcpServerName);
    }

    if (accessToken) {
      headers['Authorization'] = `Bearer ${accessToken}`;
    }
  }

  const transportOptions = {
    requestInit: { headers: { ...mcpServerConfig.headers, ...headers } },
    authProvider,
  };

  return createUrlTransport(mcpServerName, mcpServerConfig, transportOptions);
}
```

### 7.5 401 錯誤處理與自動認證

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 1417-1663)

```typescript
try {
  // 首次嘗試連接
  const transport = await createTransport(/*...*/);
  await mcpClient.connect(transport, { timeout });
  return mcpClient;
} catch (initialError) {
  // 檢查是否為 401 認證錯誤
  if (isAuthenticationError(initialError) && hasNetworkTransport(mcpServerConfig)) {
    // 標記此伺服器需要 OAuth
    mcpServerRequiresOAuth.set(mcpServerName, true);

    // 嘗試從錯誤提取 WWW-Authenticate header
    const wwwAuthenticate = extractWWWAuthenticateHeader(String(initialError));

    if (wwwAuthenticate) {
      // 自動 OAuth 發現與認證
      const oauthSuccess = await handleAutomaticOAuth(
        mcpServerName, mcpServerConfig, wwwAuthenticate
      );

      if (oauthSuccess) {
        // 使用新 Token 重試連接
        const accessToken = await getStoredOAuthToken(mcpServerName);
        await retryWithOAuth(mcpClient, mcpServerName, mcpServerConfig, accessToken);
        return mcpClient;
      }
    }

    // 無法自動處理，提示使用者手動認證
    throw new UnauthorizedError(
      `MCP server '${mcpServerName}' requires authentication using: /mcp auth ${mcpServerName}`
    );
  }

  throw initialError;
}
```

---

## 8. MCP 工具發現與註冊

### 8.1 工具發現流程

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 893-960)

```typescript
export async function discoverTools(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  mcpClient: Client,
  cliConfig: Config,
  messageBus?: MessageBus,
  options?: { timeout?: number; signal?: AbortSignal },
): Promise<DiscoveredMCPTool[]> {
  try {
    // 1. 檢查伺服器是否支援工具能力
    if (mcpClient.getServerCapabilities()?.tools == null) {
      return [];
    }

    // 2. 呼叫 tools/list RPC
    const response = await mcpClient.listTools({}, options);
    const discoveredTools: DiscoveredMCPTool[] = [];

    // 3. 處理每個工具定義
    for (const toolDef of response.tools) {
      try {
        // 3a. 檢查工具是否啟用 (includeTools/excludeTools)
        if (!isEnabled(toolDef, mcpServerName, mcpServerConfig)) {
          continue;
        }

        // 3b. 創建可呼叫的工具包裝器
        const mcpCallableTool = new McpCallableTool(
          mcpClient,
          toolDef,
          mcpServerConfig.timeout ?? MCP_DEFAULT_TIMEOUT_MSEC,
        );

        // 3c. 創建 DiscoveredMCPTool
        const tool = new DiscoveredMCPTool(
          mcpCallableTool,
          mcpServerName,
          toolDef.name,
          toolDef.description ?? '',
          toolDef.inputSchema ?? { type: 'object', properties: {} },
          mcpServerConfig.trust,
          undefined,
          cliConfig,
          mcpServerConfig.extension?.name,
          mcpServerConfig.extension?.id,
          messageBus,
        );

        discoveredTools.push(tool);
      } catch (error) {
        coreEvents.emitFeedback('error',
          `Error discovering tool '${toolDef.name}' from '${mcpServerName}'`, error);
      }
    }

    return discoveredTools;
  } catch (error) {
    if (!error.message?.includes('Method not found')) {
      coreEvents.emitFeedback('error',
        `Error discovering tools from ${mcpServerName}`, error);
    }
    return [];
  }
}
```

### 8.2 工具過濾邏輯

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 1822-1846)

```typescript
export function isEnabled(
  funcDecl: { name?: string },
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
): boolean {
  // 1. 檢查工具名稱是否存在
  if (!funcDecl.name) {
    debugLogger.warn(`Discovered tool without name from '${mcpServerName}'`);
    return false;
  }

  const { includeTools, excludeTools } = mcpServerConfig;

  // 2. excludeTools 優先 (黑名單)
  if (excludeTools && excludeTools.includes(funcDecl.name)) {
    return false;
  }

  // 3. includeTools 過濾 (白名單)
  // 支援完整名稱或帶參數的名稱格式
  return (
    !includeTools ||
    includeTools.some(
      (tool) =>
        tool === funcDecl.name ||
        tool.startsWith(`${funcDecl.name}(`)
    )
  );
}
```

### 8.3 McpCallableTool 工具呼叫

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 962-1023)

```typescript
class McpCallableTool implements CallableTool {
  constructor(
    private readonly client: Client,
    private readonly toolDef: McpTool,
    private readonly timeout: number,
  ) {}

  /**
   * 返回工具定義
   */
  async tool(): Promise<Tool> {
    return {
      functionDeclarations: [
        {
          name: this.toolDef.name,
          description: this.toolDef.description,
          parametersJsonSchema: this.toolDef.inputSchema,
        },
      ],
    };
  }

  /**
   * 執行工具呼叫
   */
  async callTool(functionCalls: FunctionCall[]): Promise<Part[]> {
    if (functionCalls.length !== 1) {
      throw new Error('McpCallableTool only supports single function call');
    }

    const call = functionCalls[0];

    try {
      // 呼叫 MCP 伺服器的工具
      const result = await this.client.callTool(
        {
          name: call.name!,
          arguments: call.args as Record<string, unknown>,
        },
        undefined,
        { timeout: this.timeout },
      );

      return [
        {
          functionResponse: {
            name: call.name,
            response: result,
          },
        },
      ];
    } catch (error) {
      // 返回錯誤格式化的回應
      return [
        {
          functionResponse: {
            name: call.name,
            response: {
              error: {
                message: error instanceof Error ? error.message : String(error),
                isError: true,
              },
            },
          },
        },
      ];
    }
  }
}
```

### 8.4 動態工具更新 (Coalescing Pattern)

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 278-316, 391-451)

```typescript
/**
 * 註冊通知處理器
 */
private registerNotificationHandlers(): void {
  if (!this.client) return;

  const capabilities = this.client.getServerCapabilities();

  // 監聽工具列表變更
  if (capabilities?.tools?.listChanged) {
    this.client.setNotificationHandler(
      ToolListChangedNotificationSchema,
      async () => {
        debugLogger.log(`Received tool update notification from '${this.serverName}'`);
        await this.refreshTools();
      },
    );
  }

  // 監聽資源列表變更
  if (capabilities?.resources?.listChanged) {
    this.client.setNotificationHandler(
      ResourceListChangedNotificationSchema,
      async () => {
        await this.refreshResources();
      },
    );
  }
}

/**
 * 刷新工具 (Coalescing Pattern)
 * 合併快速連續的更新請求，避免競爭條件
 */
private async refreshTools(): Promise<void> {
  // 如果正在刷新，標記為待處理
  if (this.isRefreshingTools) {
    this.pendingToolRefresh = true;
    return;
  }

  this.isRefreshingTools = true;

  try {
    do {
      this.pendingToolRefresh = false;

      if (this.status !== MCPServerStatus.CONNECTED) break;

      // 發現新工具
      const newTools = await this.discoverTools(this.cliConfig);

      // 移除舊工具並註冊新工具
      this.toolRegistry.removeMcpToolsByServer(this.serverName);
      for (const tool of newTools) {
        this.toolRegistry.registerTool(tool);
      }
      this.toolRegistry.sortTools();

      // 通知工具更新回調
      if (this.onToolsUpdated) {
        await this.onToolsUpdated();
      }

    } while (this.pendingToolRefresh); // 處理待處理的更新
  } finally {
    this.isRefreshingTools = false;
    this.pendingToolRefresh = false;
  }
}
```

---

## 9. Token 儲存實現

### 9.1 儲存架構概覽

系統採用分層的 Token 儲存架構：

```
┌─────────────────────────────────────────────────────────────────┐
│                      MCPOAuthTokenStorage                       │
│                     (統一介面層)                                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      HybridTokenStorage                         │
│                     (自動選擇策略)                               │
│                                                                 │
│  ┌──────────────────────┐     ┌──────────────────────────────┐ │
│  │                      │     │                              │ │
│  │  KeychainTokenStorage│◀───▶│   FileTokenStorage           │ │
│  │  (主要: OS Keychain) │     │   (回退: AES-256-GCM 加密)   │ │
│  │                      │     │                              │ │
│  └──────────────────────┘     └──────────────────────────────┘ │
│            ▲                              ▲                     │
│            │                              │                     │
│    使用 keytar 模組              使用 Node.js crypto           │
│    (macOS Keychain,              ~/.gemini/mcp-oauth-tokens   │
│     Windows Credential,          -v2.json (加密)               │
│     Linux Secret Service)                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 TokenStorage 介面

**檔案路徑:** `/packages/core/src/mcp/token-storage/types.ts` (Lines 30-50)

```typescript
export interface TokenStorage {
  getCredentials(serverName: string): Promise<OAuthCredentials | null>;
  setCredentials(credentials: OAuthCredentials): Promise<void>;
  deleteCredentials(serverName: string): Promise<void>;
  listServers(): Promise<string[]>;
  getAllCredentials(): Promise<Map<string, OAuthCredentials>>;
  clearAll(): Promise<void>;
}

export interface SecretStorage {
  setSecret(key: string, value: string): Promise<void>;
  getSecret(key: string): Promise<string | null>;
  deleteSecret(key: string): Promise<void>;
  listSecrets(): Promise<string[]>;
}

export enum TokenStorageType {
  KEYCHAIN = 'keychain',
  ENCRYPTED_FILE = 'encrypted_file',
}
```

### 9.3 HybridTokenStorage 自動選擇

**檔案路徑:** `/packages/core/src/mcp/token-storage/hybrid-token-storage.ts` (Lines 14-97)

```typescript
export class HybridTokenStorage extends BaseTokenStorage {
  private storage: TokenStorage | null = null;
  private storageType: TokenStorageType | null = null;
  private storageInitPromise: Promise<TokenStorage> | null = null;

  /**
   * 初始化儲存後端
   * 優先使用 Keychain，失敗時回退到加密檔案
   */
  private async initializeStorage(): Promise<TokenStorage> {
    // 環境變數強制使用加密檔案
    const forceFileStorage = process.env['GEMINI_FORCE_FILE_STORAGE'] === 'true';

    if (!forceFileStorage) {
      try {
        const { KeychainTokenStorage } = await import('./keychain-token-storage.js');
        const keychainStorage = new KeychainTokenStorage(this.serviceName);

        // 執行可用性測試
        const isAvailable = await keychainStorage.isAvailable();
        if (isAvailable) {
          this.storage = keychainStorage;
          this.storageType = TokenStorageType.KEYCHAIN;
          return this.storage;
        }
      } catch (_e) {
        // Keychain 不可用，回退
      }
    }

    // 使用加密檔案儲存
    this.storage = new FileTokenStorage(this.serviceName);
    this.storageType = TokenStorageType.ENCRYPTED_FILE;
    return this.storage;
  }

  /**
   * 取得儲存實例 (延遲初始化)
   */
  private async getStorage(): Promise<TokenStorage> {
    if (this.storage !== null) return this.storage;

    // 使用單一 Promise 避免競爭條件
    if (!this.storageInitPromise) {
      this.storageInitPromise = this.initializeStorage();
    }

    return this.storageInitPromise;
  }

  // 代理所有操作到實際儲存
  async getCredentials(serverName: string): Promise<OAuthCredentials | null> {
    const storage = await this.getStorage();
    return storage.getCredentials(serverName);
  }

  // ... 其他方法類似
}
```

### 9.4 Keychain Token 儲存

**檔案路徑:** `/packages/core/src/mcp/token-storage/keychain-token-storage.ts` (Lines 28-335)

```typescript
export class KeychainTokenStorage extends BaseTokenStorage implements SecretStorage {
  private keychainAvailable: boolean | null = null;
  private keytarModule: Keytar | null = null;
  private keytarLoadAttempted = false;

  /**
   * 動態載入 keytar 模組
   */
  async getKeytar(): Promise<Keytar | null> {
    if (this.keytarLoadAttempted) {
      return this.keytarModule;
    }

    this.keytarLoadAttempted = true;

    try {
      const moduleName = 'keytar';
      const module = await import(moduleName);
      this.keytarModule = module.default || module;
    } catch (_) {
      // keytar 是可選的，不拋出錯誤
    }

    return this.keytarModule;
  }

  /**
   * 測試 Keychain 是否可用
   */
  async checkKeychainAvailability(): Promise<boolean> {
    if (this.keychainAvailable !== null) {
      return this.keychainAvailable;
    }

    try {
      const keytar = await this.getKeytar();
      if (!keytar) {
        this.keychainAvailable = false;
        return false;
      }

      // 執行完整的設定-取得-刪除測試
      const testAccount = `__keychain_test__${crypto.randomBytes(8).toString('hex')}`;
      const testPassword = 'test';

      await keytar.setPassword(this.serviceName, testAccount, testPassword);
      const retrieved = await keytar.getPassword(this.serviceName, testAccount);
      const deleted = await keytar.deletePassword(this.serviceName, testAccount);

      const success = deleted && retrieved === testPassword;
      this.keychainAvailable = success;
      return success;
    } catch (_error) {
      this.keychainAvailable = false;
      return false;
    }
  }

  /**
   * 儲存憑證到 Keychain
   */
  async setCredentials(credentials: OAuthCredentials): Promise<void> {
    if (!(await this.checkKeychainAvailability())) {
      throw new Error('Keychain is not available');
    }

    const keytar = await this.getKeytar();
    if (!keytar) throw new Error('Keytar module not available');

    this.validateCredentials(credentials);

    const sanitizedName = this.sanitizeServerName(credentials.serverName);
    const updatedCredentials: OAuthCredentials = {
      ...credentials,
      updatedAt: Date.now(),
    };

    // 序列化為 JSON 並儲存
    const data = JSON.stringify(updatedCredentials);
    await keytar.setPassword(this.serviceName, sanitizedName, data);
  }

  /**
   * 從 Keychain 取得憑證
   */
  async getCredentials(serverName: string): Promise<OAuthCredentials | null> {
    if (!(await this.checkKeychainAvailability())) {
      throw new Error('Keychain is not available');
    }

    const keytar = await this.getKeytar();
    if (!keytar) throw new Error('Keytar module not available');

    const sanitizedName = this.sanitizeServerName(serverName);
    const data = await keytar.getPassword(this.serviceName, sanitizedName);

    if (!data) return null;

    const credentials = JSON.parse(data) as OAuthCredentials;

    // 檢查 Token 是否過期
    if (this.isTokenExpired(credentials)) {
      return null;
    }

    return credentials;
  }
}
```

### 9.5 加密檔案 Token 儲存

**檔案路徑:** `/packages/core/src/mcp/token-storage/file-token-storage.ts` (Lines 15-185)

```typescript
export class FileTokenStorage extends BaseTokenStorage {
  private readonly tokenFilePath: string;
  private readonly encryptionKey: Buffer;

  constructor(serviceName: string) {
    super(serviceName);
    const configDir = path.join(os.homedir(), '.gemini');
    this.tokenFilePath = path.join(configDir, 'mcp-oauth-tokens-v2.json');
    this.encryptionKey = this.deriveEncryptionKey();
  }

  /**
   * 從機器特定資訊導出加密金鑰
   * 使用 scrypt KDF (PBKDF)
   */
  private deriveEncryptionKey(): Buffer {
    const salt = `${os.hostname()}-${os.userInfo().username}-gemini-cli`;
    return crypto.scryptSync('gemini-cli-oauth', salt, 32);
  }

  /**
   * AES-256-GCM 加密
   */
  private encrypt(text: string): string {
    // 1. 生成隨機 IV (16 bytes)
    const iv = crypto.randomBytes(16);

    // 2. 創建加密器
    const cipher = crypto.createCipheriv('aes-256-gcm', this.encryptionKey, iv);

    // 3. 加密資料
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    // 4. 取得認證標籤
    const authTag = cipher.getAuthTag();

    // 5. 組合格式: iv:authTag:ciphertext
    return iv.toString('hex') + ':' + authTag.toString('hex') + ':' + encrypted;
  }

  /**
   * AES-256-GCM 解密
   */
  private decrypt(encryptedData: string): string {
    const parts = encryptedData.split(':');
    if (parts.length !== 3) {
      throw new Error('Invalid encrypted data format');
    }

    // 1. 解析各部分
    const iv = Buffer.from(parts[0], 'hex');
    const authTag = Buffer.from(parts[1], 'hex');
    const encrypted = parts[2];

    // 2. 創建解密器
    const decipher = crypto.createDecipheriv('aes-256-gcm', this.encryptionKey, iv);
    decipher.setAuthTag(authTag);

    // 3. 解密資料
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }

  /**
   * 確保目錄存在且權限正確
   */
  private async ensureDirectoryExists(): Promise<void> {
    const dir = path.dirname(this.tokenFilePath);
    await fs.mkdir(dir, { recursive: true, mode: 0o700 }); // 僅擁有者可存取
  }

  /**
   * 載入並解密所有 Token
   */
  private async loadTokens(): Promise<Map<string, OAuthCredentials>> {
    try {
      const data = await fs.readFile(this.tokenFilePath, 'utf-8');
      const decrypted = this.decrypt(data);
      const tokens = JSON.parse(decrypted) as Record<string, OAuthCredentials>;
      return new Map(Object.entries(tokens));
    } catch (error: unknown) {
      const err = error as NodeJS.ErrnoException;
      if (err.code === 'ENOENT') {
        return new Map(); // 檔案不存在
      }
      if (err.message?.includes('Invalid encrypted data format') ||
          err.message?.includes('Unable to authenticate data')) {
        throw new Error('Token file corrupted');
      }
      throw error;
    }
  }

  /**
   * 加密並儲存所有 Token
   */
  private async saveTokens(tokens: Map<string, OAuthCredentials>): Promise<void> {
    await this.ensureDirectoryExists();

    const data = Object.fromEntries(tokens);
    const json = JSON.stringify(data, null, 2);
    const encrypted = this.encrypt(json);

    // 設定嚴格的檔案權限 (僅擁有者讀寫)
    await fs.writeFile(this.tokenFilePath, encrypted, { mode: 0o600 });
  }

  async setCredentials(credentials: OAuthCredentials): Promise<void> {
    this.validateCredentials(credentials);

    const tokens = await this.loadTokens();
    tokens.set(credentials.serverName, {
      ...credentials,
      updatedAt: Date.now(),
    });
    await this.saveTokens(tokens);
  }
}
```

#### 加密細節表

| 項目 | 值 | 說明 |
|------|-----|------|
| **演算法** | AES-256-GCM | 認證加密，同時提供機密性和完整性 |
| **金鑰長度** | 256 bits (32 bytes) | 由 scrypt 導出 |
| **IV 長度** | 128 bits (16 bytes) | 每次加密隨機生成 |
| **Auth Tag** | 128 bits (16 bytes) | GCM 認證標籤 |
| **KDF** | scrypt | `scrypt('gemini-cli-oauth', salt, 32)` |
| **Salt** | `{hostname}-{username}-gemini-cli` | 機器特定 |
| **儲存格式** | `{iv_hex}:{authTag_hex}:{ciphertext_hex}` | 冒號分隔的 hex 編碼 |
| **檔案權限** | `0o600` | 僅擁有者讀寫 |
| **檔案位置** | `~/.gemini/mcp-oauth-tokens-v2.json` | 使用者家目錄下 |

---

## 10. 安全考量

### 10.1 OAuth 安全機制

| 安全機制 | 實現方式 | 目的 |
|----------|----------|------|
| **PKCE** | S256 challenge (`SHA256(verifier).base64url()`) | 防止授權碼攔截攻擊 |
| **State 參數** | 16 bytes 隨機值，callback 驗證 | 防止 CSRF 攻擊 |
| **重定向 URI 驗證** | 限制為 localhost | 防止 Token 洩漏到惡意網站 |
| **動態埠號** | `port 0` 讓 OS 分配，或 `OAUTH_CALLBACK_PORT` | 避免埠號衝突 |
| **Callback 超時** | 5 分鐘 | 限制攻擊窗口 |
| **Resource 參數** | RFC 9728 resource 驗證 | 確保 Token 只用於預期的資源 |

### 10.2 Token 安全處理

| 安全措施 | 實現方式 | 說明 |
|----------|----------|------|
| **安全儲存** | OS Keychain (優先) 或 AES-256-GCM 加密檔案 | 保護靜態 Token |
| **過期緩衝** | 5 分鐘預緩衝 | 避免使用即將過期的 Token |
| **自動刷新** | 過期前自動使用 refresh_token 刷新 | 無縫續期 |
| **無效移除** | 刷新失敗時自動刪除無效 Token | 清理過期憑證 |
| **Token 指紋** | `SHA256(token).hex().slice(0, 8)` | 日誌記錄時不洩漏完整 Token |
| **檔案權限** | `0o600` (僅擁有者讀寫) | 限制檔案存取 |

### 10.3 環境安全

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 1789-1807)

```typescript
// 淨化環境變數後傳遞給 MCP 子程序
env: {
  ...sanitizeEnvironment(process.env, sanitizationConfig),
  ...(mcpServerConfig.env || {}),
}
```

環境變數淨化配置：

```typescript
interface EnvironmentSanitizationConfig {
  allowedEnvironmentVariables: string[];    // 允許傳遞的變數白名單
  blockedEnvironmentVariables: string[];    // 阻止傳遞的變數黑名單
  enableEnvironmentVariableRedaction: boolean; // 是否啟用敏感變數遮蔽
}
```

### 10.4 主機白名單驗證

Google Credentials 提供者只允許連接到特定主機：

```typescript
// 檔案: /packages/core/src/mcp/google-auth-provider.ts
const ALLOWED_HOSTS = [
  /^.+\.googleapis\.com$/,     // Google API 端點
  /^(.*\.)?luci\.app$/         // LUCI 應用
];
```

### 10.5 安全最佳實踐清單

```
✓ PKCE 授權碼流程 (RFC 7636)
✓ State 參數 CSRF 保護
✓ 動態客戶端註冊 (RFC 7591)
✓ 受保護資源元資料驗證 (RFC 9728)
✓ Token 加密儲存 (AES-256-GCM)
✓ OS Keychain 整合
✓ 自動 Token 刷新
✓ 過期 Token 自動清理
✓ 環境變數淨化
✓ 主機白名單驗證
✓ 檔案權限限制 (0o600)
✓ Token 指紋日誌 (避免洩漏)
✓ Callback 超時保護
✓ 安全瀏覽器啟動
```

---

## 11. 連接與認證流程圖

### 11.1 完整 OAuth 認證流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MCPOAuthProvider.authenticate()                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 1: OAuth 發現                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1a. 發送 HEAD 請求到 MCP 伺服器                                      │   │
│  │     檢查回應狀態 (401/307)                                           │   │
│  │     提取 WWW-Authenticate header                                     │   │
│  └────────────────────────────────┬────────────────────────────────────┘   │
│                                   ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1b. 從 WWW-Authenticate 發現 OAuth 配置                              │   │
│  │     解析 resource_metadata URL                                       │   │
│  │     獲取受保護資源元資料 (RFC 9728)                                   │   │
│  │     驗證 resource 參數匹配                                           │   │
│  └────────────────────────────────┬────────────────────────────────────┘   │
│                                   ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1c. 發現授權伺服器元資料 (RFC 8414)                                   │   │
│  │     嘗試多種 well-known 端點                                         │   │
│  │     獲取 authorization_endpoint, token_endpoint, registration_endpoint│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 2: PKCE 生成                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────┐                                     │
│  │ code_verifier = random(32 bytes)   │ → 43 字元 base64url                 │
│  │ code_challenge = SHA256(verifier)  │ → base64url 編碼                    │
│  │ state = random(16 bytes)           │ → CSRF 保護                         │
│  └────────────────────────────────────┘                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 3: Callback 伺服器                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────┐                                     │
│  │ 監聽 port 0 (OS 分配)              │                                     │
│  │ 或 OAUTH_CALLBACK_PORT 環境變數    │                                     │
│  │ 路徑: /oauth/callback              │                                     │
│  │ 超時: 5 分鐘                        │                                     │
│  └────────────────────────────────────┘                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 4: 動態客戶端註冊 (如果沒有 clientId)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ POST {registration_endpoint}                                        │    │
│  │ {                                                                   │    │
│  │   "client_name": "Gemini CLI MCP Client",                          │    │
│  │   "redirect_uris": ["http://localhost:{port}/oauth/callback"],     │    │
│  │   "grant_types": ["authorization_code", "refresh_token"],          │    │
│  │   "response_types": ["code"],                                       │    │
│  │   "token_endpoint_auth_method": "none"                              │    │
│  │ }                                                                   │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                              ↓                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ 回應: { client_id, client_secret? }                                 │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 5: 建構授權 URL                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  {authorization_endpoint}?                                                  │
│    client_id={client_id}                                                    │
│    response_type=code                                                       │
│    redirect_uri=http://localhost:{port}/oauth/callback                      │
│    state={state}                                                            │
│    code_challenge={code_challenge}                                          │
│    code_challenge_method=S256                                               │
│    scope={scopes}                                                           │
│    audience={audiences}                                                     │
│    resource={mcp_server_url}  ← RFC 9728                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 6: 開啟瀏覽器                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  使用 openBrowserSecurely() 安全啟動瀏覽器                                   │
│  顯示 URL 供手動複製 (如果自動開啟失敗)                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
           ┌──────────────────────────┴──────────────────────────┐
           │                    使用者在瀏覽器中                    │
           │                    完成授權                           │
           └──────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 7: 處理 Callback                                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ 接收: /oauth/callback?code={code}&state={state}                     │    │
│  │ 驗證 state 參數 (防止 CSRF)                                          │    │
│  │ 提取授權碼                                                          │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 8: 交換 Token                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ POST {token_endpoint}                                               │    │
│  │ Content-Type: application/x-www-form-urlencoded                     │    │
│  │                                                                     │    │
│  │ grant_type=authorization_code                                       │    │
│  │ code={code}                                                         │    │
│  │ redirect_uri=http://localhost:{port}/oauth/callback                 │    │
│  │ code_verifier={code_verifier}  ← PKCE 驗證                          │    │
│  │ client_id={client_id}                                               │    │
│  │ client_secret={client_secret}  ← 如果有                             │    │
│  │ audience={audiences}                                                │    │
│  │ resource={mcp_server_url}                                           │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                              ↓                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ 回應: {                                                             │    │
│  │   access_token,                                                     │    │
│  │   token_type,                                                       │    │
│  │   expires_in,                                                       │    │
│  │   refresh_token,                                                    │    │
│  │   scope                                                             │    │
│  │ }                                                                   │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 9: 儲存 Token                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ HybridTokenStorage.setCredentials({                                 │    │
│  │   serverName,                                                       │    │
│  │   token: { accessToken, refreshToken, expiresAt, tokenType, scope },│    │
│  │   clientId,                                                         │    │
│  │   tokenUrl,                                                         │    │
│  │   mcpServerUrl,                                                     │    │
│  │   updatedAt                                                         │    │
│  │ })                                                                  │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  儲存至:                                                                     │
│  ├── Keychain (如果可用)                                                     │
│  └── ~/.gemini/mcp-oauth-tokens-v2.json (AES-256-GCM 加密)                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  階段 10: 驗證儲存                                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  讀取儲存的 Token                                                            │
│  計算 Token 指紋: SHA256(accessToken).hex().slice(0, 8)                     │
│  記錄驗證結果 (不記錄完整 Token)                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │ 返回 OAuthToken │
                              └───────────────┘
```

### 11.2 MCP 連接流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        connectToMcpServer()                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  創建 MCP Client                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  const mcpClient = new Client({                                             │
│    name: 'gemini-cli-mcp-client',                                           │
│    version: '0.0.1'                                                         │
│  }, {                                                                       │
│    jsonSchemaValidator: new LenientJsonSchemaValidator()                    │
│  });                                                                        │
│                                                                             │
│  註冊 roots 能力                                                             │
│  設定 ListRootsRequest 處理器                                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  選擇傳輸層                                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│    httpUrl?  ──yes──▶  StreamableHTTPClientTransport                        │
│       │                                                                     │
│       no                                                                    │
│       │                                                                     │
│       ▼                                                                     │
│    url + type:'http'?  ──yes──▶  StreamableHTTPClientTransport              │
│       │                                                                     │
│       no                                                                    │
│       │                                                                     │
│       ▼                                                                     │
│    url + type:'sse'?  ──yes──▶  SSEClientTransport                          │
│       │                                                                     │
│       no                                                                    │
│       │                                                                     │
│       ▼                                                                     │
│    url (無 type)?  ──yes──▶  StreamableHTTPClientTransport (預設)           │
│       │                                                                     │
│       no                                                                    │
│       │                                                                     │
│       ▼                                                                     │
│    command?  ──yes──▶  StdioClientTransport                                 │
│       │                                                                     │
│       no                                                                    │
│       │                                                                     │
│       ▼                                                                     │
│    拋出錯誤: Invalid configuration                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  首次連接嘗試                                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  try {                                                                      │
│    await mcpClient.connect(transport, { timeout });                         │
│    return mcpClient; // 成功                                                 │
│  } catch (error) {                                                          │
│    // 處理錯誤...                                                            │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
         401 錯誤              其他錯誤                 成功
              │                       │                       │
              ▼                       ▼                       ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐
│  OAuth 認證流程       │  │  SSE 回退嘗試         │  │  返回 mcpClient   │
├──────────────────────┤  ├──────────────────────┤  └──────────────────┘
│                      │  │  (如果是 HTTP 失敗    │
│ 1. 標記需要 OAuth     │  │   且未指定 type)      │
│                      │  │                      │
│ 2. 提取 WWW-Auth     │  │ try {                │
│    header            │  │   SSEClientTransport │
│                      │  │   connect()          │
│ 3. OAuth 配置發現     │  │ } catch {           │
│                      │  │   // 也失敗          │
│ 4. 觸發認證流程       │  │ }                    │
│                      │  │                      │
│ 5. 獲取 Token        │  └──────────┬───────────┘
│                      │             │
│ 6. 重試連接          │     ┌───────┴───────┐
│                      │     │               │
└──────────┬───────────┘  401 錯誤      其他錯誤
           │                 │               │
           ▼                 ▼               ▼
┌──────────────────────┐ ┌────────────┐ ┌────────────┐
│  retryWithOAuth()    │ │  進入      │ │  拋出      │
├──────────────────────┤ │  OAuth 流程 │ │  原始錯誤   │
│                      │ └────────────┘ └────────────┘
│ if (httpReturned404) │
│   connectWithSSE()   │
│ else                 │
│   createHTTPTransport│
│   connect()          │
│   if (404)           │
│     connectWithSSE() │
│                      │
└──────────┬───────────┘
           │
           ▼
    ┌─────────────┐
    │ 返回        │
    │ mcpClient   │
    └─────────────┘
```

---

## 關鍵檔案索引

| 檔案路徑 | 行數 | 主要功能 |
|----------|------|----------|
| `/packages/core/src/mcp/oauth-provider.ts` | 1032 | OAuth 2.0 + PKCE + RFC 9728 實現 |
| `/packages/core/src/mcp/oauth-utils.ts` | 431 | OAuth 發現工具 (RFC 8414, RFC 9728) |
| `/packages/core/src/tools/mcp-client.ts` | 1847 | MCP 客戶端、連接、發現、傳輸 |
| `/packages/core/src/tools/mcp-client-manager.ts` | 358 | MCP 客戶端管理器 |
| `/packages/core/src/mcp/google-auth-provider.ts` | 158 | Google ADC 提供者 |
| `/packages/core/src/mcp/sa-impersonation-provider.ts` | 157 | 服務帳戶模擬提供者 |
| `/packages/core/src/mcp/oauth-token-storage.ts` | 235 | OAuth Token 管理層 |
| `/packages/core/src/mcp/token-storage/types.ts` | 50 | Token 儲存介面定義 |
| `/packages/core/src/mcp/token-storage/hybrid-token-storage.ts` | 98 | 混合 Token 儲存 |
| `/packages/core/src/mcp/token-storage/keychain-token-storage.ts` | 336 | OS Keychain 整合 |
| `/packages/core/src/mcp/token-storage/file-token-storage.ts` | 186 | AES-256-GCM 加密檔案儲存 |
| `/packages/core/src/mcp/auth-provider.ts` | 19 | MCP 認證提供者介面 |
| `/packages/core/src/config/config.ts` | 1806 | 配置定義 (MCPServerConfig, AuthProviderType) |
| `/packages/core/src/core/contentGenerator.ts` | 200+ | 主要認證類型 (AuthType) |

---

## 總結

Gemini CLI 的認證與 MCP 系統採用了多層次、模組化的架構設計：

1. **認證多樣性**: 支援 5 種主要認證類型和 3 種 MCP 專用認證提供者，涵蓋個人使用者、企業用戶和自動化環境。

2. **OAuth 標準遵循**: 完整實現 OAuth 2.0 (RFC 6749)、PKCE (RFC 7636)、動態客戶端註冊 (RFC 7591)、授權伺服器元資料 (RFC 8414) 和受保護資源元資料 (RFC 9728)。

3. **安全儲存**: 採用 OS Keychain 優先、AES-256-GCM 加密檔案回退的混合策略，確保 Token 安全儲存。

4. **自動化流程**: Token 自動刷新、OAuth 自動發現、認證錯誤自動處理，提供無縫的使用者體驗。

5. **傳輸層抽象**: 支援 Stdio、SSE、HTTP Streamable 三種傳輸類型，適應各種 MCP 伺服器部署場景。

6. **工具動態管理**: 支援工具的動態發現、過濾、更新，以及 Coalescing Pattern 處理快速連續的更新通知。

此架構設計兼顧了安全性、靈活性和易用性，為 Gemini CLI 與各種 MCP 伺服器的整合提供了堅實的基礎。
