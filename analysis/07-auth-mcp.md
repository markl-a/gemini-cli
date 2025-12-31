# 認證與 MCP 系統深度分析報告

## 1. 認證方法

### 1.1 主要認證類型

**檔案路徑:** `/packages/core/src/core/contentGenerator.ts` (Lines 49-55)

```typescript
export enum AuthType {
  LOGIN_WITH_GOOGLE = 'oauth-personal',      // OAuth2 Google 登入
  USE_GEMINI = 'gemini-api-key',             // Gemini API Key
  USE_VERTEX_AI = 'vertex-ai',               // Vertex AI 憑證
  LEGACY_CLOUD_SHELL = 'cloud-shell',        // Cloud Shell ADC
  COMPUTE_ADC = 'compute-default-credentials', // Compute Engine ADC
}
```

### 1.2 MCP 特定認證提供者

**檔案路徑:** `/packages/core/src/config/config.ts` (Lines 247-251)

```typescript
export enum AuthProviderType {
  DYNAMIC_DISCOVERY = 'dynamic_discovery',           // 自動發現 OAuth
  GOOGLE_CREDENTIALS = 'google_credentials',         // Google ADC
  SERVICE_ACCOUNT_IMPERSONATION = 'service_account_impersonation',
}
```

### 1.3 OAuth 配置介面

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 25-36)

```typescript
export interface MCPOAuthConfig {
  enabled?: boolean;
  clientId?: string;
  clientSecret?: string;
  authorizationUrl?: string;
  tokenUrl?: string;
  scopes?: string[];
  audiences?: string[];
  redirectUri?: string;
  tokenParamName?: string;
  registrationUrl?: string;
}
```

---

## 2. Token 管理與刷新

### 2.1 Token 儲存資料結構

**檔案路徑:** `/packages/core/src/mcp/token-storage/types.ts` (Lines 10-28)

```typescript
export interface OAuthToken {
  accessToken: string;
  refreshToken?: string;
  expiresAt?: number;  // 毫秒，包含 5 分鐘緩衝
  tokenType: string;   // 例如 'Bearer'
  scope?: string;
}

export interface OAuthCredentials {
  serverName: string;
  token: OAuthToken;
  clientId?: string;
  tokenUrl?: string;
  mcpServerUrl?: string;
  updatedAt: number;
}
```

### 2.2 Token 刷新實現

**檔案路徑:** `/packages/core/src/mcp/oauth-provider.ts` (Lines 960-1030)

**getValidToken() 方法處理:**
1. 檢查 token 是否過期 (包含 5 分鐘緩衝)
2. 如果有 refresh_token 則嘗試刷新
3. 在過期前自動刷新
4. 刷新失敗時移除無效 token
5. 無有效 token 時返回 null

**Token 刷新流程 (Lines 587-698):**
```
refreshAccessToken() → POST 到 token 端點:
  - grant_type: 'refresh_token'
  - refresh_token: <stored_refresh_token>
  - client_id: <config_clientId>
  - client_secret: <optional>
  - scopes: <requested_scopes>
  - audience: <optional>
  - resource: <MCP server URL for RFC 9728>
```

### 2.3 Token 過期緩衝

**檔案路徑:** `/packages/core/src/mcp/oauth-utils.ts` (Line 51)

```typescript
export const FIVE_MIN_BUFFER_MS = 5 * 60 * 1000;
```

---

## 3. 認證提供者

### 3.1 Google Credentials 提供者

**檔案路徑:** `/packages/core/src/mcp/google-auth-provider.ts`

- 使用 `google-auth-library` 的 `GoogleAuth`
- 從 Application Default Credentials (ADC) 獲取 token
- 實現快取與過期時間檢查
- 驗證允許的主機:
  ```typescript
  const ALLOWED_HOSTS = [/^.+\.googleapis\.com$/, /^(.*\.)?luci\.app$/];
  ```
- 透過 `X-Goog-User-Project` header 返回配額專案 ID

### 3.2 服務帳戶模擬提供者

**檔案路徑:** `/packages/core/src/mcp/sa-impersonation-provider.ts`

- 使用 IAM Credentials API 模擬目標服務帳戶
- 必要配置:
  - `targetServiceAccount`: SA email
  - `targetAudience`: OAuth Client ID
  - `url` 或 `httpUrl`: MCP server endpoint
- Token 生成流程 (Lines 76-138):
  1. 檢查快取 token 有效性
  2. 呼叫 IAM API 生成 ID token
  3. 從 ID token 解析 JWT 過期時間
  4. 返回 ID token 作為 Bearer token

### 3.3 OAuth 提供者 (RFC 6749 + RFC 9728)

**檔案路徑:** `/packages/core/src/mcp/oauth-provider.ts` (Lines 99-1031)

**主要特性:**
- PKCE 支援 (Proof Key for Code Exchange) - RFC 7636
- 動態客戶端註冊 - RFC 7591
- RFC 9728 (Protected Resource) 元資料發現
- State 參數驗證防止 CSRF
- 自動瀏覽器開啟
- 動態埠號本地 HTTP callback 伺服器

---

## 4. OAuth 認證流程

**Lines 709-950:**

```
authenticate() →
  1. OAuth 發現 (如果未提供 URL)
     - 檢查 WWW-Authenticate header
     - 從 well-known 端點發現 OAuth 配置
     - 嘗試 path-based 和 root-based 發現

  2. 生成 PKCE 參數
     - codeVerifier: 43-128 字元隨機字串
     - codeChallenge: SHA256(codeVerifier).base64url()
     - state: 16 位元組隨機值 (CSRF 保護)

  3. 啟動 callback 伺服器
     - 監聽 port 0 (OS 分配) 或 OAUTH_CALLBACK_PORT
     - 5 分鐘超時

  4. 動態客戶端註冊 (如果沒有 clientId)
     - POST 到 registration_endpoint
     - 從授權伺服器元資料發現註冊端點

  5. 建構授權 URL:
     - code_challenge, code_challenge_method=S256
     - scope, audience, resource (RFC 9728)
     - state 參數

  6. 開啟瀏覽器並等待 callback

  7. 交換 code 獲取 tokens
     - POST with code, code_verifier, client_id, redirect_uri

  8. 儲存 token
```

---

## 5. MCP 客戶端實現

### 5.1 MCP 客戶端架構

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts`

**核心類別:**
- `McpClient` (Lines 107-451): 單一 MCP 伺服器連接
- `McpClientManager`: 管理多個客戶端
- `McpCallableTool` (Lines 962-1023): 包裝 MCP 工具

**MCP 伺服器狀態列舉 (Lines 78-87):**
```typescript
enum MCPServerStatus {
  DISCONNECTED = 'disconnected',
  CONNECTING = 'connecting',
  CONNECTED = 'connected',
  DISCONNECTING = 'disconnecting',
}
```

### 5.2 MCP 伺服器配置

**檔案路徑:** `/packages/core/src/config/config.ts` (Lines 208-244)

```typescript
export class MCPServerConfig {
  // stdio 傳輸
  readonly command?: string;
  readonly args?: string[];
  readonly env?: Record<string, string>;
  readonly cwd?: string;

  // SSE 傳輸
  readonly url?: string;

  // HTTP 傳輸 (已棄用)
  readonly httpUrl?: string;
  readonly headers?: Record<string, string>;

  // 傳輸類型指定
  readonly type?: 'sse' | 'http';

  // 通用
  readonly timeout?: number;
  readonly trust?: boolean;

  // 工具過濾
  readonly includeTools?: string[];
  readonly excludeTools?: string[];

  // 認證
  readonly oauth?: MCPOAuthConfig;
  readonly authProviderType?: AuthProviderType;
  readonly targetAudience?: string;
  readonly targetServiceAccount?: string;
}
```

---

## 6. 傳輸層

### 6.1 傳輸選擇邏輯

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 1718-1815)

**優先順序:**
1. `httpUrl` (已棄用) → `StreamableHTTPClientTransport`
2. `url` + `type: 'http'` → `StreamableHTTPClientTransport`
3. `url` + `type: 'sse'` → `SSEClientTransport`
4. `url` 無類型 → `StreamableHTTPClientTransport` (預設)
5. `command` → `StdioClientTransport`

### 6.2 Stdio 傳輸 (子程序)

**Lines 1789-1809:**
```typescript
const transport = new StdioClientTransport({
  command: mcpServerConfig.command,
  args: mcpServerConfig.args || [],
  env: {
    ...sanitizeEnvironment(process.env, sanitizationConfig),
    ...(mcpServerConfig.env || {}),
  },
  cwd: mcpServerConfig.cwd,
  stderr: 'pipe',
});
```

- 捕獲 stderr 用於調試
- 傳遞前淨化環境變數
- 父子通訊透過 stdin/stdout

### 6.3 SSE 傳輸

**Lines 1199-1214:**
- Server-Sent Events 用於 HTTP 串流
- OAuth token 透過 `Authorization: Bearer` header 傳遞
- 支援配置中的自訂 headers

### 6.4 HTTP (Streamable) 傳輸

**Lines 1718-1715:**
- 使用類似 WebSocket 的協議
- 支援動態 header 注入的 auth providers
- 透過 `requestInit` 配置 headers

---

## 7. MCP 工具發現與註冊

### 7.1 工具發現流程

**檔案路徑:** `/packages/core/src/tools/mcp-client.ts` (Lines 893-960)

```typescript
export async function discoverTools(...) {
  // 1. 檢查伺服器是否支援工具能力
  if (mcpClient.getServerCapabilities()?.tools == null) return [];

  // 2. 呼叫 tools/list RPC
  const response = await mcpClient.listTools({}, options);

  // 3. 對每個工具:
  //    - 檢查是否啟用 (includeTools/excludeTools 過濾)
  //    - 創建 McpCallableTool 包裝器
  //    - 創建 DiscoveredMCPTool
  //    - 註冊到 toolRegistry
}
```

**工具過濾 (Lines 1822-1846):**
```typescript
export function isEnabled(funcDecl, mcpServerName, mcpServerConfig) {
  // excludeTools 優先
  if (excludeTools?.includes(funcDecl.name)) return false;

  // includeTools 是白名單
  return !includeTools || includeTools.some(...);
}
```

### 7.2 工具調用

**Lines 981-1022:**
```typescript
async callTool(functionCalls) {
  const call = functionCalls[0];

  try {
    const result = await this.client.callTool({
      name: call.name!,
      arguments: call.args,
    }, undefined, { timeout: this.timeout });

    return [{ functionResponse: { name: call.name, response: result } }];
  } catch (error) {
    return [{ functionResponse: { name: call.name, response: { error: {...} } } }];
  }
}
```

---

## 8. Token 儲存實現

### 8.1 混合 Token 儲存

**檔案路徑:** `/packages/core/src/mcp/token-storage/hybrid-token-storage.ts`

- 主要: Keychain (如果可用)
- 回退: 加密檔案儲存
- 環境變數: `GEMINI_FORCE_FILE_STORAGE=true` 強制使用檔案

### 8.2 Keychain Token 儲存

**檔案路徑:** `/packages/core/src/mcp/token-storage/keychain-token-storage.ts`

- 使用 `keytar` 模組 (可選，優雅回退)
- 服務名稱: `KEYCHAIN_SERVICE_NAME`
- 帳戶名稱: 帶前綴的淨化伺服器名稱

### 8.3 加密檔案 Token 儲存

**檔案路徑:** `/packages/core/src/mcp/token-storage/file-token-storage.ts`

**加密細節:**
- 演算法: AES-256-GCM
- 金鑰導出: `crypto.scryptSync('gemini-cli-oauth', salt, 32)`
- Salt: `{hostname}-{username}-gemini-cli`
- 格式: `{iv_hex}:{authTag_hex}:{ciphertext_hex}`
- 檔案權限: `0o600` (僅擁有者讀寫)
- 位置: `~/.gemini/mcp-oauth-tokens-v2.json`

---

## 9. 安全考量

### 9.1 認證安全

| 方面 | 實現 |
|------|------|
| PKCE 流程 | SHA256(codeVerifier).base64url() |
| State 參數 | 16 位元組隨機，callback 驗證防止 CSRF |
| 重定向驗證 | 可配置 redirect_uri，預設 localhost |
| 超時 | 5 分鐘 callback 超時 |
| 埠號分配 | OS 分配 (port 0) 或環境變數 |
| Token 儲存 | AES-256-GCM 加密或 OS Keychain |
| Token 指紋 | SHA256 雜湊 (前 8 字元) 記錄 |

### 9.2 Token 處理

| 方面 | 實現 |
|------|------|
| 過期緩衝 | 5 分鐘預緩衝 |
| 刷新儲存 | access_token 和 refresh_token 都快取 |
| 自動刷新 | 有 refresh_token 時過期前觸發 |
| 無效移除 | 過期/失敗 token 自動從儲存刪除 |
| 選擇性記錄 | Token 指紋記錄，從不記錄完整值 |

### 9.3 環境安全

**Lines 1789-1807:**
```typescript
env: {
  ...sanitizeEnvironment(process.env, sanitizationConfig),
  ...(mcpServerConfig.env || {}),
}
```

- 生成 MCP 程序前移除敏感環境變數
- 透過 `EnvironmentSanitizationConfig` 可配置

---

## 10. 連接與認證流程圖

### OAuth 認證流程

```
MCPOAuthProvider.authenticate()
├─ 1. OAuth 發現
│  ├─ 檢查 WWW-Authenticate header
│  ├─ 嘗試 OAuth 伺服器元資料發現
│  └─ 解析 OAuth 配置
├─ 2. PKCE 生成
│  ├─ codeVerifier (43-128 字元)
│  ├─ codeChallenge = SHA256(codeVerifier).base64url()
│  └─ state (16 位元組隨機)
├─ 3. Callback 伺服器
│  └─ 監聽動態埠號
├─ 4. 客戶端註冊 (如果沒有 clientId)
│  └─ POST 客戶端註冊請求
├─ 5. 建構授權 URL
│  └─ 添加 code_challenge, state, scopes 等
├─ 6. 開啟瀏覽器
│  └─ 使用者授權並重定向到 callback
├─ 7. 交換 Code
│  └─ POST code + code_verifier → access_token
├─ 8. Token 儲存
│  └─ 儲存到 Keychain 或加密檔案
└─ 返回 token
```

### MCP 連接流程

```
connectToMcpServer()
├─ 創建 MCP Client
├─ 嘗試主要傳輸
│  ├─ HTTP (StreamableHTTPClientTransport)
│  ├─ SSE (SSEClientTransport)
│  └─ Stdio (StdioClientTransport)
├─ 401 錯誤時
│  ├─ 提取 www-authenticate header
│  ├─ 發現 OAuth 配置
│  ├─ 呼叫 authenticate()
│  ├─ 使用 Bearer token 重試
│  └─ HTTP 返回 404 時回退到 SSE
└─ 成功 → 返回 Client
```

---

## 關鍵檔案摘要

| 檔案 | 行數 | 用途 |
|------|------|------|
| `oauth-provider.ts` | 1032 | OAuth 2.0 + RFC 9728 實現 |
| `mcp-client.ts` | 1847+ | MCP 客戶端連接、發現、傳輸 |
| `google-auth-provider.ts` | 158 | Google ADC 提供者 |
| `sa-impersonation-provider.ts` | 157 | 服務帳戶模擬 |
| `file-token-storage.ts` | 186 | AES-256-GCM 加密 token 儲存 |
| `keychain-token-storage.ts` | 200+ | OS Keychain 整合 |
| `oauth-token-storage.ts` | 235 | OAuth token 管理器 |
| `oauth-utils.ts` | 431 | OAuth 發現 (RFC 8414, RFC 9728) |
