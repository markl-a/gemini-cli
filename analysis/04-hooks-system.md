# Hooks 系統深度分析報告

> 本報告深入分析 Gemini CLI 的 Hooks 系統架構，涵蓋所有核心元件、執行流程、安全機制及設計模式。

---

## 目錄

1. [系統架構概覽](#1-系統架構概覽)
2. [Hook 註冊表](#2-hook-註冊表)
3. [Hook 類型與事件系統](#3-hook-類型與事件系統)
4. [Hook 輸入/輸出架構](#4-hook-輸入輸出架構)
5. [Hook 執行器](#5-hook-執行器)
6. [Hook 聚合器](#6-hook-聚合器)
7. [Hook 轉換器](#7-hook-轉換器)
8. [執行流程序列圖](#8-執行流程序列圖)
9. [整合點](#9-整合點)
10. [安全考量](#10-安全考量)
11. [關鍵設計模式](#11-關鍵設計模式)

---

## 1. 系統架構概覽

### 1.1 高階架構圖

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              HookSystem (協調器)                                  │
│                                                                                  │
│  主要職責：                                                                       │
│  - 初始化所有 Hook 相關元件                                                        │
│  - 提供統一的 Hook 管理介面                                                        │
│  - 協調元件間的依賴關係                                                           │
└─────────────┬──────────────────────────────────────────────────────┬─────────────┘
              │                                                      │
              │ 依賴注入                                              │ 事件觸發
              │                                                      │
    ┌─────────▼─────────────┐                             ┌──────────▼──────────────┐
    │   HookRegistry        │                             │   HookEventHandler      │
    │                       │                             │                         │
    │   職責：               │                             │   職責：                 │
    │   - 配置載入           │                             │   - 接收事件觸發請求     │
    │   - Hook 發現          │                             │   - 創建事件輸入         │
    │   - 來源追蹤           │                             │   - 協調執行流程         │
    │   - 啟用/禁用管理      │                             │   - 處理 MessageBus      │
    └─────────┬─────────────┘                             └──────────┬──────────────┘
              │                                                      │
              │ 提供 Hook 條目                                        │ 執行協調
              │                                                      │
    ┌─────────▼─────────────┐                             ┌──────────▼──────────────┐
    │   HookPlanner         │◄────────────────────────────►│   HookRunner            │
    │                       │     執行計劃                  │                         │
    │   職責：               │                             │   職責：                 │
    │   - 上下文匹配         │                             │   - Shell 程序生成       │
    │   - 執行計劃創建       │                             │   - 命令執行             │
    │   - Hook 去重          │                             │   - 輸出收集             │
    │   - 執行策略決定       │                             │   - 超時管理             │
    └─────────┬─────────────┘                             └──────────┬──────────────┘
              │                                                      │
              └──────────────────────┬───────────────────────────────┘
                                    │
                                    │ 執行結果
                                    │
                          ┌─────────▼─────────────┐
                          │   HookAggregator      │
                          │                       │
                          │   職責：               │
                          │   - 結果合併           │
                          │   - 策略選擇           │
                          │   - 決策聚合           │
                          │   - 錯誤收集           │
                          └─────────┬─────────────┘
                                    │
                                    │ 聚合結果
                                    │
                          ┌─────────▼─────────────┐
                          │   HookTranslator      │
                          │                       │
                          │   職責：               │
                          │   - SDK 類型轉換       │
                          │   - 版本相容           │
                          │   - 格式標準化         │
                          └───────────────────────┘
```

### 1.2 元件職責詳細說明

| 元件 | 檔案位置 | 主要職責 | 關鍵方法 |
|------|----------|----------|----------|
| **HookSystem** | `hookSystem.ts` | 主協調器，初始化並管理所有元件 | `initialize()`, `getEventHandler()`, `getAllHooks()` |
| **HookRegistry** | `hookRegistry.ts` | 從多個配置來源載入、驗證和發現 hooks | `initialize()`, `getHooksForEvent()`, `setHookEnabled()` |
| **HookPlanner** | `hookPlanner.ts` | 根據上下文匹配 hooks 並創建執行計劃 | `createExecutionPlan()`, `matchesContext()` |
| **HookRunner** | `hookRunner.ts` | 在 shell 環境中執行命令 hooks | `executeHook()`, `executeHooksParallel()`, `executeHooksSequential()` |
| **HookAggregator** | `hookAggregator.ts` | 使用事件特定策略合併多個 hooks 的結果 | `aggregateResults()`, `mergeOutputs()` |
| **HookTranslator** | `hookTranslator.ts` | 在 SDK 類型和 hook 穩定格式之間轉換 | `toHookLLMRequest()`, `fromHookLLMResponse()` |
| **HookEventHandler** | `hookEventHandler.ts` | 協調所有元件並觸發各類事件 | `fireBeforeToolEvent()`, `fireAfterModelEvent()` |

### 1.3 元件初始化流程

```typescript
// HookSystem 建構函式
export class HookSystem {
  constructor(config: Config) {
    const logger: Logger = logs.getLogger(SERVICE_NAME);
    const messageBus = config.getMessageBus();

    // 1. 初始化註冊表 (獨立元件)
    this.hookRegistry = new HookRegistry(config);

    // 2. 初始化執行器 (獨立元件)
    this.hookRunner = new HookRunner(config);

    // 3. 初始化聚合器 (無依賴)
    this.hookAggregator = new HookAggregator();

    // 4. 初始化計劃器 (依賴註冊表)
    this.hookPlanner = new HookPlanner(this.hookRegistry);

    // 5. 初始化事件處理器 (依賴所有其他元件)
    this.hookEventHandler = new HookEventHandler(
      config,
      logger,
      this.hookPlanner,
      this.hookRunner,
      this.hookAggregator,
      messageBus,  // 可選的 MessageBus 支援
    );
  }
}
```

---

## 2. Hook 註冊表

### 2.1 配置來源優先順序

```
優先順序 (由高到低)：

┌─────────────────┐
│  1. Project     │  ← 專案級別 (.gemini/settings.json)
│     優先順序: 1   │     最高優先權，但需要信任資料夾
└────────┬────────┘
         │
┌────────▼────────┐
│  2. User        │  ← 使用者級別 (~/.gemini/settings.json)
│     優先順序: 2   │     個人配置
└────────┬────────┘
         │
┌────────▼────────┐
│  3. System      │  ← 系統級別 (/etc/gemini/settings.json)
│     優先順序: 3   │     全系統配置
└────────┬────────┘
         │
┌────────▼────────┐
│  4. Extensions  │  ← 擴展來源
│     優先順序: 4   │     從 MCP 擴展載入
└─────────────────┘
```

### 2.2 配置來源優先順序實作

```typescript
// 來源優先順序數值對應
private getSourcePriority(source: ConfigSource): number {
  switch (source) {
    case ConfigSource.Project:    return 1;  // 最高優先
    case ConfigSource.User:       return 2;
    case ConfigSource.System:     return 3;
    case ConfigSource.Extensions: return 4;  // 最低優先
    default:                      return 999;
  }
}
```

### 2.3 關鍵資料結構

#### 2.3.1 Hook 註冊條目

```typescript
/**
 * 帶完整元資料的註冊表條目
 * 每個已註冊的 Hook 都用此結構表示
 */
interface HookRegistryEntry {
  config: HookConfig;           // 實際 hook 配置 (命令、超時等)
  source: ConfigSource;         // 來源 (Project/User/System/Extensions)
  eventName: HookEventName;     // 監聽的事件名稱
  matcher?: string;             // 模式匹配器 (工具名或觸發器)
  sequential?: boolean;         // 是否要求順序執行
  enabled: boolean;             // 啟用/禁用標記
}
```

#### 2.3.2 Hook 配置結構

```typescript
/**
 * 命令 Hook 配置
 */
interface CommandHookConfig {
  type: HookType.Command;       // Hook 類型標識
  command: string;              // 要執行的 Shell 命令
  name?: string;                // 可選友好名稱 (用於識別和禁用)
  description?: string;         // 可選描述
  timeout?: number;             // 最大執行時間 (毫秒, 預設 60000)
  source?: ConfigSource;        // 註冊時填充的來源資訊
}

/**
 * Hook 定義 (包含匹配器和多個 hooks)
 */
interface HookDefinition {
  matcher?: string;             // 正規表達式匹配模式
  sequential?: boolean;         // 是否順序執行
  hooks: HookConfig[];          // 實際的 hook 配置陣列
}
```

### 2.4 註冊表初始化流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    HookRegistry.initialize()                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 清空現有條目                                                  │
│     this.entries = [];                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 檢查專案 Hook 信任                                            │
│     if (this.config.isTrustedFolder()) {                        │
│       this.checkProjectHooksTrust();                            │
│     }                                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 處理專案級 Hooks (如果資料夾已信任)                            │
│     const configHooks = this.config.getHooks();                 │
│     if (configHooks && this.config.isTrustedFolder()) {         │
│       this.processHooksConfiguration(configHooks, Project);     │
│     }                                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 處理擴展 Hooks                                                │
│     for (const extension of extensions) {                       │
│       if (extension.isActive && extension.hooks) {              │
│         this.processHooksConfiguration(hooks, Extensions);      │
│       }                                                         │
│     }                                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 記錄初始化結果                                                │
│     debugLogger.log(`Hook registry initialized with             │
│       ${this.entries.length} hook entries`);                    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.5 Hook 禁用機制

```typescript
// 設定檔中的 disabled 陣列
{
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "write_file",
        "hooks": [
          { "type": "command", "command": "node security-check.js" },
          { "type": "command", "command": "node audit-log.js" }
        ]
      }
    ],
    "disabled": ["node audit-log.js"]  // 透過命令字串禁用特定 hook
  }
}

// 禁用檢查邏輯
private processHookDefinition(definition, eventName, source): void {
  const disabledHooks = this.config.getDisabledHooks() || [];

  for (const hookConfig of definition.hooks) {
    const hookName = this.getHookName({ config: hookConfig });
    const isDisabled = disabledHooks.includes(hookName);

    this.entries.push({
      config: hookConfig,
      source,
      eventName,
      matcher: definition.matcher,
      sequential: definition.sequential,
      enabled: !isDisabled,  // 根據禁用列表設定
    });
  }
}
```

---

## 3. Hook 類型與事件系統

### 3.1 支援的 Hook 事件 (11 種)

```typescript
enum HookEventName {
  // === 工具相關事件 ===
  BeforeTool = 'BeforeTool',                    // 工具執行前
  AfterTool = 'AfterTool',                      // 工具執行後
  BeforeToolSelection = 'BeforeToolSelection',  // 工具選擇邏輯前

  // === 代理相關事件 ===
  BeforeAgent = 'BeforeAgent',                  // 代理提示處理前
  AfterAgent = 'AfterAgent',                    // 代理回應後

  // === 模型相關事件 ===
  BeforeModel = 'BeforeModel',                  // LLM API 呼叫前
  AfterModel = 'AfterModel',                    // LLM API 呼叫後

  // === 會話生命週期事件 ===
  SessionStart = 'SessionStart',                // 會話初始化
  SessionEnd = 'SessionEnd',                    // 會話終止

  // === 其他事件 ===
  Notification = 'Notification',                // 通知事件 (如權限請求)
  PreCompress = 'PreCompress',                  // 歷史壓縮前
}
```

### 3.2 事件詳細說明表

| 事件名稱 | 觸發時機 | 典型用途 | 可修改內容 |
|---------|---------|---------|-----------|
| **BeforeTool** | 工具執行前 | 安全檢查、輸入驗證、稽核日誌 | 阻擋執行、修改工具輸入 |
| **AfterTool** | 工具執行後 | 結果過濾、額外上下文添加 | 添加 additionalContext |
| **BeforeToolSelection** | 工具選擇前 | 限制可用工具、強制工具選擇 | 修改 toolConfig |
| **BeforeAgent** | 代理處理前 | 提示增強、安全注入 | 添加 additionalContext |
| **AfterAgent** | 代理回應後 | 回應審核、紀錄 | 處理 stop_hook_active |
| **BeforeModel** | LLM 呼叫前 | 請求修改、模擬回應 | 修改 llm_request、提供合成 llm_response |
| **AfterModel** | LLM 回應後 | 回應過濾、內容修改 | 修改 llm_response |
| **SessionStart** | 會話開始 | 環境初始化、歡迎訊息 | 添加 additionalContext |
| **SessionEnd** | 會話結束 | 清理工作、紀錄摘要 | 無 |
| **Notification** | 通知觸發 | 權限請求處理、自定義通知 | suppressOutput、systemMessage |
| **PreCompress** | 壓縮前 | 備份、自定義壓縮 | 無 |

### 3.3 事件觸發來源詳解

#### SessionStart 來源

```typescript
enum SessionStartSource {
  Startup = 'startup',   // 應用程式首次啟動
  Resume = 'resume',     // 恢復先前會話
  Clear = 'clear',       // /clear 命令後重新開始
}
```

#### SessionEnd 原因

```typescript
enum SessionEndReason {
  Exit = 'exit',                    // 正常退出
  Clear = 'clear',                  // /clear 命令
  Logout = 'logout',                // 登出操作
  PromptInputExit = 'prompt_input_exit',  // 輸入中斷
  Other = 'other',                  // 其他原因
}
```

#### PreCompress 觸發器

```typescript
enum PreCompressTrigger {
  Manual = 'manual',    // 手動觸發 (/compress 命令)
  Auto = 'auto',        // 自動觸發 (達到 token 閾值)
}
```

#### Notification 類型

```typescript
enum NotificationType {
  ToolPermission = 'ToolPermission',  // 工具權限請求
}
```

---

## 4. Hook 輸入/輸出架構

### 4.1 基礎輸入結構

```typescript
/**
 * 所有 Hook 事件的基礎輸入
 * 每個事件都會包含這些共用欄位
 */
interface HookInput {
  session_id: string;           // 當前會話的唯一識別碼
  transcript_path: string;      // 對話歷史檔案路徑
  cwd: string;                  // 當前工作目錄
  hook_event_name: string;      // 觸發的事件名稱
  timestamp: string;            // ISO 8601 格式時間戳
}
```

### 4.2 事件特定輸入結構

```typescript
// === BeforeTool 輸入 ===
interface BeforeToolInput extends HookInput {
  tool_name: string;                    // 將要執行的工具名稱
  tool_input: Record<string, unknown>;  // 工具的輸入參數
}

// === AfterTool 輸入 ===
interface AfterToolInput extends HookInput {
  tool_name: string;                    // 已執行的工具名稱
  tool_input: Record<string, unknown>;  // 工具的輸入參數
  tool_response: Record<string, unknown>; // 工具的執行結果
}

// === BeforeAgent 輸入 ===
interface BeforeAgentInput extends HookInput {
  prompt: string;                       // 使用者的原始提示
}

// === AfterAgent 輸入 ===
interface AfterAgentInput extends HookInput {
  prompt: string;                       // 原始提示
  prompt_response: string;              // 代理的回應
  stop_hook_active: boolean;            // 是否有 stop hook 啟用
}

// === BeforeModel 輸入 ===
interface BeforeModelInput extends HookInput {
  llm_request: LLMRequest;              // 穩定格式的 LLM 請求
}

// === AfterModel 輸入 ===
interface AfterModelInput extends HookInput {
  llm_request: LLMRequest;              // 原始 LLM 請求
  llm_response: LLMResponse;            // LLM 的回應
}

// === Notification 輸入 ===
interface NotificationInput extends HookInput {
  notification_type: NotificationType;  // 通知類型
  message: string;                      // 通知訊息
  details: Record<string, unknown>;     // 通知詳情
}

// === Session 輸入 ===
interface SessionStartInput extends HookInput {
  source: SessionStartSource;           // 會話開始來源
}

interface SessionEndInput extends HookInput {
  reason: SessionEndReason;             // 會話結束原因
}

// === PreCompress 輸入 ===
interface PreCompressInput extends HookInput {
  trigger: PreCompressTrigger;          // 壓縮觸發方式
}
```

### 4.3 基礎輸出結構

```typescript
/**
 * 所有 Hook 事件的基礎輸出
 */
interface HookOutput {
  // === 執行控制 ===
  continue?: boolean;           // 是否繼續執行 (false = 停止整個流程)
  stopReason?: string;          // 停止原因說明

  // === 決策相關 ===
  decision?: HookDecision;      // 決策: 'ask'|'block'|'deny'|'approve'|'allow'
  reason?: string;              // 決策原因

  // === 輸出控制 ===
  suppressOutput?: boolean;     // 是否從記錄中隱藏此輸出
  systemMessage?: string;       // 顯示給使用者的系統訊息

  // === 事件特定資料 ===
  hookSpecificOutput?: Record<string, unknown>;  // 各事件的特定輸出
}

// 決策類型定義
type HookDecision = 'ask' | 'block' | 'deny' | 'approve' | 'allow' | undefined;
```

### 4.4 事件特定輸出類別

```typescript
/**
 * BeforeTool Hook 輸出
 * 可用於修改工具輸入或阻擋執行
 */
class BeforeToolHookOutput extends DefaultHookOutput {
  getModifiedToolInput(): Record<string, unknown> | undefined {
    if (this.hookSpecificOutput && 'tool_input' in this.hookSpecificOutput) {
      return this.hookSpecificOutput['tool_input'] as Record<string, unknown>;
    }
    return undefined;
  }
}

/**
 * BeforeModel Hook 輸出
 * 可用於修改 LLM 請求或提供合成回應
 */
class BeforeModelHookOutput extends DefaultHookOutput {
  // 取得合成的 LLM 回應 (跳過實際 API 呼叫)
  getSyntheticResponse(): GenerateContentResponse | undefined {
    if (this.hookSpecificOutput && 'llm_response' in this.hookSpecificOutput) {
      const hookResponse = this.hookSpecificOutput['llm_response'] as LLMResponse;
      return defaultHookTranslator.fromHookLLMResponse(hookResponse);
    }
    return undefined;
  }

  // 修改 LLM 請求
  override applyLLMRequestModifications(
    target: GenerateContentParameters,
  ): GenerateContentParameters {
    if (this.hookSpecificOutput && 'llm_request' in this.hookSpecificOutput) {
      const hookRequest = this.hookSpecificOutput['llm_request'] as Partial<LLMRequest>;
      const sdkRequest = defaultHookTranslator.fromHookLLMRequest(hookRequest as LLMRequest, target);
      return { ...target, ...sdkRequest };
    }
    return target;
  }
}

/**
 * BeforeToolSelection Hook 輸出
 * 可用於修改工具配置
 */
class BeforeToolSelectionHookOutput extends DefaultHookOutput {
  override applyToolConfigModifications(target: {
    toolConfig?: GenAIToolConfig;
    tools?: ToolListUnion;
  }): { toolConfig?: GenAIToolConfig; tools?: ToolListUnion } {
    if (this.hookSpecificOutput && 'toolConfig' in this.hookSpecificOutput) {
      const hookToolConfig = this.hookSpecificOutput['toolConfig'] as HookToolConfig;
      const sdkToolConfig = defaultHookTranslator.fromHookToolConfig(hookToolConfig);
      return { ...target, tools: target.tools || [], toolConfig: sdkToolConfig };
    }
    return target;
  }
}

/**
 * AfterModel Hook 輸出
 * 可用於修改 LLM 回應
 */
class AfterModelHookOutput extends DefaultHookOutput {
  getModifiedResponse(): GenerateContentResponse | undefined {
    if (this.hookSpecificOutput && 'llm_response' in this.hookSpecificOutput) {
      const hookResponse = this.hookSpecificOutput['llm_response'] as Partial<LLMResponse>;
      if (hookResponse?.candidates?.[0]?.content?.parts?.length) {
        return defaultHookTranslator.fromHookLLMResponse(hookResponse as LLMResponse);
      }
    }

    // 如果 hook 要求停止執行，創建合成的停止回應
    if (this.shouldStopExecution()) {
      const stopResponse: LLMResponse = {
        candidates: [{
          content: {
            role: 'model',
            parts: [this.getEffectiveReason() || 'Execution stopped by hook'],
          },
          finishReason: 'STOP',
        }],
      };
      return defaultHookTranslator.fromHookLLMResponse(stopResponse);
    }

    return undefined;
  }
}
```

### 4.5 輸出工廠函式

```typescript
/**
 * 根據事件名稱創建適當的輸出類別實例
 */
function createHookOutput(eventName: string, data: Partial<HookOutput>): DefaultHookOutput {
  switch (eventName) {
    case 'BeforeModel':
      return new BeforeModelHookOutput(data);
    case 'AfterModel':
      return new AfterModelHookOutput(data);
    case 'BeforeToolSelection':
      return new BeforeToolSelectionHookOutput(data);
    case 'BeforeTool':
      return new BeforeToolHookOutput(data);
    default:
      return new DefaultHookOutput(data);
  }
}
```

---

## 5. Hook 執行器

### 5.1 執行模式

#### 5.1.1 並行執行 (預設)

```typescript
/**
 * 並行執行多個 hooks
 * 適用於相互獨立的 hooks
 */
async executeHooksParallel(
  hookConfigs: HookConfig[],
  eventName: HookEventName,
  input: HookInput,
): Promise<HookExecutionResult[]> {
  // 使用 Promise.all 同時執行所有 hooks
  const promises = hookConfigs.map((config) =>
    this.executeHook(config, eventName, input)
  );
  return Promise.all(promises);
}
```

#### 5.1.2 順序執行

```typescript
/**
 * 順序執行多個 hooks
 * 當 sequential: true 時使用
 * 前一個 hook 的輸出會影響下一個 hook 的輸入
 */
async executeHooksSequential(
  hookConfigs: HookConfig[],
  eventName: HookEventName,
  input: HookInput,
): Promise<HookExecutionResult[]> {
  const results: HookExecutionResult[] = [];
  let currentInput = input;

  for (const config of hookConfigs) {
    const result = await this.executeHook(config, eventName, currentInput);
    results.push(result);

    // 如果 hook 成功且有輸出，修改下一個 hook 的輸入
    if (result.success && result.output) {
      currentInput = this.applyHookOutputToInput(currentInput, result.output, eventName);
    }
  }

  return results;
}
```

### 5.2 單一 Hook 執行流程

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           executeHook() 執行流程                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

                    開始執行
                       │
                       ▼
        ┌──────────────────────────────┐
        │  1. 安全檢查                  │
        │     - 檢查是否為專案 hook     │
        │     - 檢查資料夾是否已信任    │
        └──────────────┬───────────────┘
                       │
           ┌───────────┴───────────┐
           │                       │
           ▼                       ▼
    ┌─────────────┐         ┌──────────────────┐
    │ 已信任      │         │ 未信任            │
    │ (繼續執行)  │         │ (返回錯誤結果)    │
    └──────┬──────┘         └──────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  2. 命令變數展開                  │
│     $GEMINI_PROJECT_DIR → cwd    │
│     $CLAUDE_PROJECT_DIR → cwd    │
│     (使用 shell 適當的轉義)      │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  3. Shell 配置                    │
│     - Windows: PowerShell        │
│     - Unix/Linux/macOS: Bash     │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  4. 環境變數準備                  │
│     - 淨化敏感變數               │
│     - 添加 GEMINI_PROJECT_DIR    │
│     - 添加 CLAUDE_PROJECT_DIR    │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  5. 生成子程序                    │
│     spawn(executable, args, {    │
│       env: sanitizedEnv,         │
│       cwd: input.cwd,            │
│       stdio: ['pipe','pipe',     │
│               'pipe'],           │
│       shell: false               │
│     })                           │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  6. 發送輸入                      │
│     - 將 HookInput 轉為 JSON     │
│     - 寫入 stdin                 │
│     - 關閉 stdin                 │
└──────────────┬───────────────────┘
               │
     ┌─────────┴─────────┐
     │                   │
     ▼                   ▼
┌──────────┐      ┌───────────────┐
│ 收集     │      │ 設定超時計時器│
│ stdout   │      │ (預設 60 秒)  │
│ stderr   │      └───────┬───────┘
└────┬─────┘              │
     │                    │
     │    ┌───────────────┘
     │    │
     ▼    ▼
┌──────────────────────────────────┐
│  7. 等待程序結束                  │
│     - 清除超時計時器             │
│     - 計算執行時間               │
└──────────────┬───────────────────┘
               │
               │ 超時?
     ┌─────────┴─────────┐
     │ 否                │ 是
     ▼                   ▼
┌──────────────┐   ┌──────────────────┐
│ 8. 解析輸出  │   │ 發送 SIGTERM     │
│    (見下方)  │   │ 5秒後 SIGKILL    │
└──────┬───────┘   │ 返回超時錯誤     │
       │           └──────────────────┘
       ▼
┌──────────────────────────────────┐
│  9. 返回 HookExecutionResult     │
│     {                            │
│       hookConfig, eventName,     │
│       success, output,           │
│       stdout, stderr,            │
│       exitCode, duration,        │
│       error                      │
│     }                            │
└──────────────────────────────────┘
```

### 5.3 輸出解析邏輯

```typescript
// 解析 hook 輸出
let output: HookOutput | undefined;

if (exitCode === EXIT_CODE_SUCCESS && stdout.trim()) {
  try {
    let parsed = JSON.parse(stdout.trim());
    // 處理雙重編碼的 JSON 字串
    if (typeof parsed === 'string') {
      parsed = JSON.parse(parsed);
    }
    if (parsed) {
      output = parsed as HookOutput;
    }
  } catch {
    // 非 JSON，轉換純文字為結構化輸出
    output = this.convertPlainTextToHookOutput(stdout.trim(), exitCode);
  }
} else if (exitCode !== EXIT_CODE_SUCCESS && stderr.trim()) {
  // 將錯誤輸出轉換為結構化格式
  output = this.convertPlainTextToHookOutput(stderr.trim(), exitCode || EXIT_CODE_NON_BLOCKING_ERROR);
}
```

### 5.4 退出碼語義

| 退出碼 | 常數 | 含義 | 產生的決策 | 行為 |
|--------|------|------|-----------|------|
| **0** | `EXIT_CODE_SUCCESS` | 成功執行 | `'allow'` (或自訂 JSON) | 繼續執行流程 |
| **1** | `EXIT_CODE_NON_BLOCKING_ERROR` | 非阻擋錯誤 | `'allow'` + 警告訊息 | 繼續執行，但顯示警告 |
| **2** | `EXIT_CODE_BLOCKING_ERROR` | 阻擋錯誤 | `'deny'` + 錯誤原因 | 停止執行流程 |
| **其他** | - | 視為非阻擋錯誤 | `'allow'` + 警告訊息 | 繼續執行，但顯示警告 |

### 5.5 純文字輸出轉換

```typescript
/**
 * 將純文字輸出轉換為結構化的 HookOutput
 */
private convertPlainTextToHookOutput(text: string, exitCode: number): HookOutput {
  if (exitCode === EXIT_CODE_SUCCESS) {
    // 成功 - 視為系統訊息或額外上下文
    return {
      decision: 'allow',
      systemMessage: text,
    };
  } else if (exitCode === EXIT_CODE_BLOCKING_ERROR) {
    // 阻擋錯誤 - 拒絕執行
    return {
      decision: 'deny',
      reason: text,
    };
  } else {
    // 非阻擋錯誤 - 允許但顯示警告
    return {
      decision: 'allow',
      systemMessage: `Warning: ${text}`,
    };
  }
}
```

### 5.6 順序執行的輸入修改

```typescript
/**
 * 根據 hook 輸出修改下一個 hook 的輸入
 * 僅在順序執行模式下使用
 */
private applyHookOutputToInput(
  originalInput: HookInput,
  hookOutput: HookOutput,
  eventName: HookEventName,
): HookInput {
  const modifiedInput = { ...originalInput };

  if (hookOutput.hookSpecificOutput) {
    switch (eventName) {
      case HookEventName.BeforeAgent:
        // 將額外上下文附加到提示
        if ('additionalContext' in hookOutput.hookSpecificOutput) {
          const additionalContext = hookOutput.hookSpecificOutput['additionalContext'];
          if (typeof additionalContext === 'string' && 'prompt' in modifiedInput) {
            (modifiedInput as BeforeAgentInput).prompt += '\n\n' + additionalContext;
          }
        }
        break;

      case HookEventName.BeforeModel:
        // 合併修改後的 LLM 請求
        if ('llm_request' in hookOutput.hookSpecificOutput) {
          const currentRequest = (modifiedInput as BeforeModelInput).llm_request;
          const partialRequest = hookOutput.hookSpecificOutput.llm_request;
          (modifiedInput as BeforeModelInput).llm_request = {
            ...currentRequest,
            ...partialRequest,
          } as LLMRequest;
        }
        break;

      case HookEventName.BeforeTool:
        // 合併修改後的工具輸入
        if ('tool_input' in hookOutput.hookSpecificOutput) {
          const newToolInput = hookOutput.hookSpecificOutput['tool_input'] as Record<string, unknown>;
          if (newToolInput && 'tool_input' in modifiedInput) {
            (modifiedInput as BeforeToolInput).tool_input = {
              ...(modifiedInput as BeforeToolInput).tool_input,
              ...newToolInput,
            };
          }
        }
        break;
    }
  }

  return modifiedInput;
}
```

---

## 6. Hook 聚合器

### 6.1 聚合策略概覽

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           HookAggregator 策略選擇                              │
└───────────────────────────────────────────────────────────────────────────────┘

                              事件類型
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────┐
│   OR 邏輯       │    │  欄位替換       │    │  工具選擇 (特殊)    │
│                 │    │                 │    │                     │
│ - BeforeTool    │    │ - BeforeModel   │    │ - BeforeToolSelection│
│ - AfterTool     │    │ - AfterModel    │    │                     │
│ - BeforeAgent   │    │                 │    │                     │
│ - AfterAgent    │    │                 │    │                     │
│ - SessionStart  │    │                 │    │                     │
│                 │    │                 │    │                     │
│ 任何阻擋決策    │    │ 後面的 hook     │    │ 工具的 UNION        │
│ 都會阻擋執行    │    │ 覆蓋前面的      │    │ 模式取最嚴格        │
└─────────────────┘    └─────────────────┘    └─────────────────────┘
```

### 6.2 OR 邏輯聚合 (mergeWithOrDecision)

```typescript
/**
 * 使用 OR 邏輯合併輸出
 * 任何一個 hook 的阻擋決策都會導致整體阻擋
 */
private mergeWithOrDecision(outputs: HookOutput[]): HookOutput {
  const merged: HookOutput = {
    continue: true,
    suppressOutput: false,
  };

  const messages: string[] = [];
  const reasons: string[] = [];
  const systemMessages: string[] = [];
  const additionalContexts: string[] = [];

  let hasBlockDecision = false;
  let hasContinueFalse = false;

  for (const output of outputs) {
    // 處理 continue 標記
    if (output.continue === false) {
      hasContinueFalse = true;
      merged.continue = false;
      if (output.stopReason) {
        messages.push(output.stopReason);
      }
    }

    // 處理決策 (阻擋優先)
    const tempOutput = new DefaultHookOutput(output);
    if (tempOutput.isBlockingDecision()) {
      hasBlockDecision = true;
      merged.decision = output.decision;
    }

    // 收集訊息
    if (output.reason) reasons.push(output.reason);
    if (output.systemMessage) systemMessages.push(output.systemMessage);

    // suppressOutput: 任何 true 都獲勝
    if (output.suppressOutput) merged.suppressOutput = true;

    // 合併 hookSpecificOutput
    if (output.hookSpecificOutput) {
      merged.hookSpecificOutput = {
        ...(merged.hookSpecificOutput || {}),
        ...output.hookSpecificOutput,
      };
    }

    // 收集額外上下文
    this.extractAdditionalContext(output, additionalContexts);
  }

  // 如果沒有阻擋決策，設為允許
  if (!hasBlockDecision && !hasContinueFalse) {
    merged.decision = 'allow';
  }

  // 合併訊息
  if (messages.length > 0) merged.stopReason = messages.join('\n');
  if (reasons.length > 0) merged.reason = reasons.join('\n');
  if (systemMessages.length > 0) merged.systemMessage = systemMessages.join('\n');

  // 添加合併的額外上下文
  if (additionalContexts.length > 0) {
    merged.hookSpecificOutput = {
      ...(merged.hookSpecificOutput || {}),
      additionalContext: additionalContexts.join('\n'),
    };
  }

  return merged;
}
```

#### OR 邏輯決策合併範例

```
場景 1: 混合決策
┌─────────────────────────────────────────┐
│ Hook 1: decision='allow'                │
│ Hook 2: decision='deny'                 │
│ Hook 3: decision='allow'                │
├─────────────────────────────────────────┤
│ 結果: decision='deny' (阻擋)            │
│ 原因: 任何阻擋決策都獲勝                 │
└─────────────────────────────────────────┘

場景 2: continue 標記
┌─────────────────────────────────────────┐
│ Hook 1: continue=true                   │
│ Hook 2: continue=false, stopReason="X"  │
│ Hook 3: continue=true                   │
├─────────────────────────────────────────┤
│ 結果: continue=false, stopReason="X"    │
│ 原因: 任何 continue=false 都傳播        │
└─────────────────────────────────────────┘

場景 3: 訊息合併
┌─────────────────────────────────────────┐
│ Hook 1: systemMessage="Step 1 done"     │
│ Hook 2: systemMessage="Step 2 done"     │
├─────────────────────────────────────────┤
│ 結果: systemMessage="Step 1 done\n      │
│                       Step 2 done"      │
│ 原因: 訊息會被連接                      │
└─────────────────────────────────────────┘
```

### 6.3 欄位替換聚合 (mergeWithFieldReplacement)

```typescript
/**
 * 使用欄位替換策略合併輸出
 * 後面的 hook 輸出會覆蓋前面的
 */
private mergeWithFieldReplacement(outputs: HookOutput[]): HookOutput {
  let merged: HookOutput = {};

  for (const output of outputs) {
    // 後面的輸出覆蓋前面的
    merged = {
      ...merged,
      ...output,
      hookSpecificOutput: {
        ...merged.hookSpecificOutput,
        ...output.hookSpecificOutput,
      },
    };
  }

  return merged;
}
```

#### 欄位替換範例

```
BeforeModel 事件的欄位替換:
┌─────────────────────────────────────────┐
│ Hook 1 輸出:                            │
│   llm_request.config.temperature = 0.7  │
│   llm_request.messages = [...]          │
├─────────────────────────────────────────┤
│ Hook 2 輸出:                            │
│   llm_request.config.temperature = 0.5  │ ← 覆蓋 Hook 1
├─────────────────────────────────────────┤
│ 最終結果:                               │
│   llm_request.config.temperature = 0.5  │
│   llm_request.messages = [...]          │ ← 保留自 Hook 1
└─────────────────────────────────────────┘
```

### 6.4 工具選擇聚合 (mergeToolSelectionOutputs)

```typescript
/**
 * 合併工具選擇輸出
 *
 * 策略:
 * - 工具名稱: 所有 hooks 的 UNION
 * - 模式: 最嚴格的獲勝 (NONE > ANY > AUTO)
 * - 函數名稱排序確保確定性快取
 */
private mergeToolSelectionOutputs(
  outputs: BeforeToolSelectionOutput[],
): BeforeToolSelectionOutput {
  const merged: BeforeToolSelectionOutput = {};

  const allFunctionNames = new Set<string>();
  let hasNoneMode = false;
  let hasAnyMode = false;

  for (const output of outputs) {
    const toolConfig = output.hookSpecificOutput?.toolConfig;
    if (!toolConfig) continue;

    // 檢查模式
    if (toolConfig.mode === 'NONE') hasNoneMode = true;
    else if (toolConfig.mode === 'ANY') hasAnyMode = true;

    // 收集函數名稱 (所有 hooks 的聯合)
    if (toolConfig.allowedFunctionNames) {
      for (const name of toolConfig.allowedFunctionNames) {
        allFunctionNames.add(name);
      }
    }
  }

  // 決定最終模式和函數名稱
  let finalMode: FunctionCallingConfigMode;
  let finalFunctionNames: string[] = [];

  if (hasNoneMode) {
    // NONE 模式獲勝 - 最嚴格
    finalMode = FunctionCallingConfigMode.NONE;
    finalFunctionNames = [];
  } else if (hasAnyMode) {
    // ANY 模式 (如果沒有 NONE)
    finalMode = FunctionCallingConfigMode.ANY;
    finalFunctionNames = Array.from(allFunctionNames).sort();
  } else {
    // 預設使用 AUTO 模式
    finalMode = FunctionCallingConfigMode.AUTO;
    finalFunctionNames = Array.from(allFunctionNames).sort();
  }

  merged.hookSpecificOutput = {
    hookEventName: 'BeforeToolSelection',
    toolConfig: {
      mode: finalMode,
      allowedFunctionNames: finalFunctionNames,
    },
  };

  return merged;
}
```

#### 工具選擇聚合範例

```
場景: 多個 hooks 設定不同的工具配置

┌─────────────────────────────────────────┐
│ Hook 1:                                 │
│   mode: 'AUTO'                          │
│   allowedFunctionNames: ['read_file']   │
├─────────────────────────────────────────┤
│ Hook 2:                                 │
│   mode: 'ANY'                           │
│   allowedFunctionNames: ['write_file']  │
├─────────────────────────────────────────┤
│ 最終結果:                               │
│   mode: 'ANY' (比 AUTO 更嚴格)          │
│   allowedFunctionNames:                 │
│     ['read_file', 'write_file'] (聯合)  │
└─────────────────────────────────────────┘

場景: 有 NONE 模式時

┌─────────────────────────────────────────┐
│ Hook 1: mode='AUTO', tools=['*']        │
│ Hook 2: mode='NONE'                     │
├─────────────────────────────────────────┤
│ 最終結果:                               │
│   mode: 'NONE' (最嚴格，獲勝)           │
│   allowedFunctionNames: [] (清空)       │
└─────────────────────────────────────────┘
```

---

## 7. Hook 轉換器

### 7.1 版本相容性問題與解決方案

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              版本相容性挑戰                                      │
└─────────────────────────────────────────────────────────────────────────────────┘

問題:
┌─────────────────────┐     ┌─────────────────────┐
│ SDK Version 1.x     │     │ SDK Version 2.x     │
│                     │     │ (未來版本)          │
│ - 類型定義 A        │     │ - 類型定義 B        │
│ - API 結構 X        │     │ - API 結構 Y        │
└─────────┬───────────┘     └─────────┬───────────┘
          │                           │
          │ 都需要與 Hook 互動         │
          │                           │
          └─────────┬─────────────────┘
                    │
                    ▼
          ┌─────────────────────┐
          │     Hook 腳本       │
          │                     │
          │ 期望穩定的 API      │
          │ 跨版本相容         │
          └─────────────────────┘

解決方案: HookTranslator 抽象層

┌─────────────────────┐                    ┌─────────────────────┐
│ SDK Version 1.x     │──┐                 │     Hook 腳本       │
│ GenerateContent     │  │                 │                     │
│ Parameters          │  │  ┌───────────┐  │  期望:              │
└─────────────────────┘  ├─►│           │  │  - LLMRequest       │
                         │  │   Hook    │──►│  - LLMResponse      │
┌─────────────────────┐  │  │Translator │  │  - HookToolConfig   │
│ SDK Version 2.x     │──┘  │           │◄─│                     │
│ (未來)              │     └───────────┘  └─────────────────────┘
└─────────────────────┘
```

### 7.2 穩定的 Hook API 格式

#### 7.2.1 LLMRequest (穩定格式)

```typescript
/**
 * 穩定的 LLM 請求格式
 * 不受 SDK 版本變更影響
 */
interface LLMRequest {
  model: string;                          // 模型標識符
  messages: Array<{
    role: 'user' | 'model' | 'system';    // 訊息角色
    content: string | Array<{             // 內容 (文字或結構化)
      type: string;
      [key: string]: unknown;
    }>;
  }>;
  config?: {
    temperature?: number;                 // 溫度參數
    maxOutputTokens?: number;             // 最大輸出 tokens
    topP?: number;                        // Top-P 採樣
    topK?: number;                        // Top-K 採樣
    stopSequences?: string[];             // 停止序列
    candidateCount?: number;              // 候選數量
    presencePenalty?: number;             // 出現懲罰
    frequencyPenalty?: number;            // 頻率懲罰
    [key: string]: unknown;               // 擴展欄位
  };
  toolConfig?: HookToolConfig;            // 工具配置
}
```

#### 7.2.2 LLMResponse (穩定格式)

```typescript
/**
 * 穩定的 LLM 回應格式
 */
interface LLMResponse {
  text?: string;                          // 便捷的文字取得
  candidates: Array<{
    content: {
      role: 'model';
      parts: string[];                    // 回應片段 (僅文字)
    };
    finishReason?: 'STOP' | 'MAX_TOKENS' | 'SAFETY' | 'RECITATION' | 'OTHER';
    index?: number;
    safetyRatings?: Array<{
      category: string;
      probability: string;
      blocked?: boolean;
    }>;
  }>;
  usageMetadata?: {
    promptTokenCount?: number;
    candidatesTokenCount?: number;
    totalTokenCount?: number;
  };
}
```

#### 7.2.3 HookToolConfig (穩定格式)

```typescript
/**
 * 穩定的工具配置格式
 */
interface HookToolConfig {
  mode?: 'AUTO' | 'ANY' | 'NONE';         // 工具呼叫模式
  allowedFunctionNames?: string[];        // 允許的函數名稱列表
}
```

### 7.3 轉換器實作 (HookTranslatorGenAIv1)

```typescript
/**
 * GenAI SDK v1.x 的轉換器實作
 */
export class HookTranslatorGenAIv1 extends HookTranslator {

  /**
   * SDK 請求 → Hook 格式
   *
   * 注意: 此實作僅提取文字內容
   * 非文字部分 (圖片、函數呼叫等) 會被過濾
   * 這是為了提供簡化、穩定的 hook 介面
   */
  toHookLLMRequest(sdkRequest: GenerateContentParameters): LLMRequest {
    const messages: LLMRequest['messages'] = [];

    if (sdkRequest.contents) {
      const contents = Array.isArray(sdkRequest.contents)
        ? sdkRequest.contents
        : [sdkRequest.contents];

      for (const content of contents) {
        if (typeof content === 'string') {
          messages.push({ role: 'user', content });
        } else if (isContentWithParts(content)) {
          const role = content.role === 'model' ? 'model' as const
            : content.role === 'system' ? 'system' as const
            : 'user' as const;

          const parts = Array.isArray(content.parts) ? content.parts : [content.parts];

          // 僅提取文字部分
          const textContent = parts
            .filter(hasTextProperty)
            .map((part) => part.text)
            .join('');

          if (textContent) {
            messages.push({ role, content: textContent });
          }
        }
      }
    }

    const config = extractGenerationConfig(sdkRequest);

    return {
      model: sdkRequest.model || DEFAULT_GEMINI_FLASH_MODEL,
      messages,
      config: {
        temperature: config?.temperature,
        maxOutputTokens: config?.maxOutputTokens,
        topP: config?.topP,
        topK: config?.topK,
      },
    };
  }

  /**
   * Hook 格式 → SDK 請求
   */
  fromHookLLMRequest(
    hookRequest: LLMRequest,
    baseRequest?: GenerateContentParameters,
  ): GenerateContentParameters {
    const contents = hookRequest.messages.map((message) => ({
      role: message.role === 'model' ? 'model' : message.role,
      parts: [{
        text: typeof message.content === 'string'
          ? message.content
          : String(message.content),
      }],
    }));

    const result: GenerateContentParameters = {
      ...baseRequest,
      model: hookRequest.model,
      contents,
    };

    if (hookRequest.config) {
      const baseConfig = baseRequest ? extractGenerationConfig(baseRequest) : undefined;
      result.config = {
        ...baseConfig,
        temperature: hookRequest.config.temperature,
        maxOutputTokens: hookRequest.config.maxOutputTokens,
        topP: hookRequest.config.topP,
        topK: hookRequest.config.topK,
      } as GenerateContentParameters['config'];
    }

    return result;
  }

  /**
   * SDK 回應 → Hook 格式
   */
  toHookLLMResponse(sdkResponse: GenerateContentResponse): LLMResponse {
    return {
      text: sdkResponse.text,
      candidates: (sdkResponse.candidates || []).map((candidate) => {
        const textParts = candidate.content?.parts
          ?.filter(hasTextProperty)
          .map((part) => part.text) || [];

        return {
          content: {
            role: 'model' as const,
            parts: textParts,
          },
          finishReason: candidate.finishReason as LLMResponse['candidates'][0]['finishReason'],
          index: candidate.index,
          safetyRatings: candidate.safetyRatings?.map((rating) => ({
            category: String(rating.category || ''),
            probability: String(rating.probability || ''),
          })),
        };
      }),
      usageMetadata: sdkResponse.usageMetadata ? {
        promptTokenCount: sdkResponse.usageMetadata.promptTokenCount,
        candidatesTokenCount: sdkResponse.usageMetadata.candidatesTokenCount,
        totalTokenCount: sdkResponse.usageMetadata.totalTokenCount,
      } : undefined,
    };
  }

  /**
   * Hook 格式 → SDK 回應
   */
  fromHookLLMResponse(hookResponse: LLMResponse): GenerateContentResponse {
    return {
      text: hookResponse.text,
      candidates: hookResponse.candidates.map((candidate) => ({
        content: {
          role: 'model',
          parts: candidate.content.parts.map((part) => ({ text: part })),
        },
        finishReason: candidate.finishReason as FinishReason,
        index: candidate.index,
        safetyRatings: candidate.safetyRatings,
      })),
      usageMetadata: hookResponse.usageMetadata,
    } as GenerateContentResponse;
  }

  /**
   * SDK 工具配置 → Hook 格式
   */
  toHookToolConfig(sdkToolConfig: ToolConfig): HookToolConfig {
    return {
      mode: sdkToolConfig.functionCallingConfig?.mode as HookToolConfig['mode'],
      allowedFunctionNames: sdkToolConfig.functionCallingConfig?.allowedFunctionNames,
    };
  }

  /**
   * Hook 格式 → SDK 工具配置
   */
  fromHookToolConfig(hookToolConfig: HookToolConfig): ToolConfig {
    const functionCallingConfig: FunctionCallingConfig | undefined =
      hookToolConfig.mode || hookToolConfig.allowedFunctionNames
        ? {
            mode: hookToolConfig.mode,
            allowedFunctionNames: hookToolConfig.allowedFunctionNames,
          } as FunctionCallingConfig
        : undefined;

    return { functionCallingConfig };
  }
}

// 預設轉換器實例
export const defaultHookTranslator = new HookTranslatorGenAIv1();
```

---

## 8. 執行流程序列圖

### 8.1 BeforeTool Hook 完整執行序列

```
┌─────────┐    ┌────────────────┐    ┌───────────┐    ┌──────────┐    ┌────────────┐
│  Agent  │    │HookEventHandler│    │HookPlanner│    │HookRunner│    │HookAggregator│
└────┬────┘    └───────┬────────┘    └─────┬─────┘    └────┬─────┘    └──────┬───────┘
     │                 │                   │               │                 │
     │  工具執行請求    │                   │               │                 │
     │────────────────►│                   │               │                 │
     │                 │                   │               │                 │
     │                 │ 1. 創建 BeforeToolInput          │                 │
     │                 │ ┌─────────────────────────────┐  │                 │
     │                 │ │ session_id: "xxx"           │  │                 │
     │                 │ │ cwd: "/project"             │  │                 │
     │                 │ │ hook_event_name: "BeforeTool"│ │                 │
     │                 │ │ tool_name: "write_file"     │  │                 │
     │                 │ │ tool_input: {...}           │  │                 │
     │                 │ └─────────────────────────────┘  │                 │
     │                 │                   │               │                 │
     │                 │ 2. 創建執行計劃   │               │                 │
     │                 │──────────────────►│               │                 │
     │                 │                   │               │                 │
     │                 │                   │ 查詢註冊表     │                 │
     │                 │                   │ 匹配 toolName  │                 │
     │                 │                   │ 去重 hooks     │                 │
     │                 │                   │ 決定執行策略   │                 │
     │                 │                   │               │                 │
     │                 │   執行計劃        │               │                 │
     │                 │◄──────────────────│               │                 │
     │                 │                   │               │                 │
     │                 │ 3. 執行 Hooks     │               │                 │
     │                 │───────────────────────────────────►│                 │
     │                 │                   │               │                 │
     │                 │                   │               │ 對每個 hook:     │
     │                 │                   │               │ - 安全檢查      │
     │                 │                   │               │ - 展開命令      │
     │                 │                   │               │ - 生成程序      │
     │                 │                   │               │ - 發送 JSON     │
     │                 │                   │               │ - 收集輸出      │
     │                 │                   │               │ - 解析結果      │
     │                 │                   │               │                 │
     │                 │   執行結果[]      │               │                 │
     │                 │◄───────────────────────────────────│                 │
     │                 │                   │               │                 │
     │                 │ 4. 聚合結果       │               │                 │
     │                 │────────────────────────────────────────────────────►│
     │                 │                   │               │                 │
     │                 │                   │               │                 │ OR 邏輯合併
     │                 │                   │               │                 │ 收集錯誤
     │                 │                   │               │                 │ 創建輸出類別
     │                 │                   │               │                 │
     │                 │   AggregatedHookResult            │                 │
     │                 │◄────────────────────────────────────────────────────│
     │                 │                   │               │                 │
     │                 │ 5. 處理結果       │               │                 │
     │                 │ - 記錄遙測        │               │                 │
     │                 │ - 處理系統訊息    │               │                 │
     │                 │                   │               │                 │
     │  決策結果       │                   │               │                 │
     │◄────────────────│                   │               │                 │
     │                 │                   │               │                 │
     │ if decision='deny':                │               │                 │
     │   阻擋工具執行   │                   │               │                 │
     │ else:           │                   │               │                 │
     │   繼續執行工具   │                   │               │                 │
     │                 │                   │               │                 │
```

### 8.2 BeforeModel Hook 執行序列 (含合成回應)

```
┌──────────┐    ┌────────────────┐    ┌──────────────┐    ┌───────────┐
│LLMClient │    │HookEventHandler│    │HookTranslator│    │ LLM API   │
└────┬─────┘    └───────┬────────┘    └──────┬───────┘    └─────┬─────┘
     │                  │                    │                  │
     │ 準備 API 呼叫    │                    │                  │
     │                  │                    │                  │
     │ fireBeforeModel  │                    │                  │
     │─────────────────►│                    │                  │
     │                  │                    │                  │
     │                  │ 轉換 SDK 請求       │                  │
     │                  │───────────────────►│                  │
     │                  │                    │                  │
     │                  │   LLMRequest       │                  │
     │                  │◄───────────────────│                  │
     │                  │                    │                  │
     │                  │ 執行 BeforeModel hooks                │
     │                  │ ...                │                  │
     │                  │                    │                  │
     │                  │ 聚合結果:          │                  │
     │                  │ finalOutput 包含:   │                  │
     │                  │ - llm_request (修改後)               │
     │                  │ - llm_response (合成的，可選)        │
     │                  │                    │                  │
     │  AggregatedResult│                    │                  │
     │◄─────────────────│                    │                  │
     │                  │                    │                  │
     │ 檢查是否有合成回應                    │                  │
     │                  │                    │                  │
┌────┴────────────────────────────────────────────────────────────┐
│ if (finalOutput.getSyntheticResponse()) {                       │
│   // 跳過 API 呼叫，使用 hook 提供的合成回應                     │
│   return syntheticResponse;                                      │
│ }                                                                │
└────┬────────────────────────────────────────────────────────────┘
     │                  │                    │                  │
     │ 應用請求修改      │                    │                  │
     │                  │                    │                  │
     │ 實際 API 呼叫     │                    │                  │
     │──────────────────────────────────────────────────────────►│
     │                  │                    │                  │
     │  API 回應        │                    │                  │
     │◄──────────────────────────────────────────────────────────│
     │                  │                    │                  │
     │ fireAfterModel   │                    │                  │
     │─────────────────►│                    │                  │
     │                  │                    │                  │
```

### 8.3 MessageBus 整合流程

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           MessageBus 整合架構                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

┌────────────┐                 ┌────────────┐                 ┌─────────────────┐
│   Agent    │                 │ MessageBus │                 │HookEventHandler │
└──────┬─────┘                 └──────┬─────┘                 └────────┬────────┘
       │                              │                                │
       │  發布 HOOK_EXECUTION_REQUEST │                                │
       │─────────────────────────────►│                                │
       │                              │                                │
       │                              │  轉發請求                       │
       │                              │───────────────────────────────►│
       │                              │                                │
       │                              │                                │ 處理請求:
       │                              │                                │ 1. 驗證輸入
       │                              │                                │ 2. 添加基礎欄位
       │                              │                                │ 3. 路由到對應的
       │                              │                                │    fire*Event 方法
       │                              │                                │ 4. 執行 hooks
       │                              │                                │
       │                              │  發布 HOOK_EXECUTION_RESPONSE  │
       │                              │◄───────────────────────────────│
       │                              │                                │
       │  接收回應 (correlationId)    │                                │
       │◄─────────────────────────────│                                │
       │                              │                                │
```

```typescript
// MessageBus 請求/回應類型
interface HookExecutionRequest {
  type: MessageBusType.HOOK_EXECUTION_REQUEST;
  correlationId: string;
  eventName: HookEventName;
  input: Record<string, unknown>;  // 事件特定欄位
}

interface HookExecutionResponse {
  type: MessageBusType.HOOK_EXECUTION_RESPONSE;
  correlationId: string;
  success: boolean;
  output?: Record<string, unknown>;
  error?: Error;
}
```

---

## 9. 整合點

### 9.1 工具執行整合

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              工具執行整合流程                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

                              使用者請求工具操作
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        1. BeforeTool Hook 階段                                   │
│                                                                                  │
│  fireBeforeToolEvent(toolName, toolInput)                                        │
│                                                                                  │
│  Hook 可執行的操作:                                                               │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ A. 阻擋執行                                                                 │ │
│  │    output: { decision: 'deny', reason: '安全政策禁止此操作' }               │ │
│  │                                                                            │ │
│  │ B. 修改工具輸入                                                             │ │
│  │    output: {                                                               │ │
│  │      decision: 'allow',                                                    │ │
│  │      hookSpecificOutput: {                                                 │ │
│  │        tool_input: { file_path: 'modified.txt', ... }                     │ │
│  │      }                                                                     │ │
│  │    }                                                                       │ │
│  │                                                                            │ │
│  │ C. 停止整個代理執行                                                         │ │
│  │    output: { continue: false, stopReason: '緊急停止' }                      │ │
│  │                                                                            │ │
│  │ D. 添加系統訊息                                                             │ │
│  │    output: { decision: 'allow', systemMessage: '已記錄操作' }               │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
                                       │
                           ┌───────────┴───────────┐
                           │                       │
                   被阻擋/停止                    允許
                           │                       │
                           ▼                       ▼
                  ┌─────────────────┐    ┌─────────────────────────────────────────┐
                  │ 返回阻擋訊息    │    │              2. 執行工具                 │
                  │ 或停止執行      │    │                                         │
                  └─────────────────┘    │  使用原始或修改後的工具輸入執行          │
                                         └────────────────────┬────────────────────┘
                                                              │
                                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        3. AfterTool Hook 階段                                    │
│                                                                                  │
│  fireAfterToolEvent(toolName, toolInput, toolResponse)                           │
│                                                                                  │
│  Hook 可執行的操作:                                                               │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ A. 添加額外上下文到回應                                                      │ │
│  │    output: {                                                               │ │
│  │      hookSpecificOutput: {                                                 │ │
│  │        additionalContext: '安全掃描: 檔案內容安全'                          │ │
│  │      }                                                                     │ │
│  │    }                                                                       │ │
│  │                                                                            │ │
│  │ B. 記錄/審計操作                                                            │ │
│  │    (透過 hook 腳本寫入日誌，不需要特定輸出)                                  │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                              返回最終結果給代理
```

### 9.2 模型執行整合

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              模型執行整合流程                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

                              準備 LLM API 呼叫
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        1. BeforeModel Hook 階段                                  │
│                                                                                  │
│  fireBeforeModelEvent(llmRequest)                                                │
│                                                                                  │
│  Hook 可執行的操作:                                                               │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ A. 修改 LLM 請求                                                            │ │
│  │    output: {                                                               │ │
│  │      hookSpecificOutput: {                                                 │ │
│  │        llm_request: {                                                      │ │
│  │          messages: [{ role: 'user', content: '修改後的提示' }],             │ │
│  │          config: { temperature: 0.5 }                                      │ │
│  │        }                                                                   │ │
│  │      }                                                                     │ │
│  │    }                                                                       │ │
│  │                                                                            │ │
│  │ B. 返回合成回應 (跳過實際 API 呼叫)                                          │ │
│  │    output: {                                                               │ │
│  │      hookSpecificOutput: {                                                 │ │
│  │        llm_response: {                                                     │ │
│  │          candidates: [{ content: { role: 'model', parts: ['模擬回應'] }}]  │ │
│  │        }                                                                   │ │
│  │      }                                                                     │ │
│  │    }                                                                       │ │
│  │                                                                            │ │
│  │ C. 阻擋執行                                                                 │ │
│  │    output: { continue: false, stopReason: '請求被安全政策阻擋' }            │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
                                       │
                      ┌────────────────┼────────────────┐
                      │                │                │
                 有合成回應        允許執行          被阻擋
                      │                │                │
                      ▼                ▼                ▼
             ┌─────────────────┐ ┌──────────────┐ ┌─────────────────┐
             │ 使用合成回應    │ │ 呼叫 LLM API │ │ 返回錯誤/停止   │
             │ 跳過 API 呼叫   │ │              │ └─────────────────┘
             └────────┬────────┘ └──────┬───────┘
                      │                 │
                      └────────┬────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        2. AfterModel Hook 階段                                   │
│                                                                                  │
│  fireAfterModelEvent(llmRequest, llmResponse)                                    │
│                                                                                  │
│  Hook 可執行的操作:                                                               │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ A. 修改 LLM 回應                                                            │ │
│  │    output: {                                                               │ │
│  │      hookSpecificOutput: {                                                 │ │
│  │        llm_response: {                                                     │ │
│  │          candidates: [{                                                    │ │
│  │            content: { role: 'model', parts: ['[過濾] 回應已過濾'] },       │ │
│  │            finishReason: 'STOP'                                            │ │
│  │          }]                                                                │ │
│  │        }                                                                   │ │
│  │      }                                                                     │ │
│  │    }                                                                       │ │
│  │                                                                            │ │
│  │ B. 完全覆蓋回應                                                             │ │
│  │    (透過 hookSpecificOutput.llm_response 替換整個回應)                      │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                           返回最終回應給代理
```

### 9.3 工具選擇整合

```typescript
// BeforeToolSelection Hook 的典型配置
{
  "hooks": {
    "BeforeToolSelection": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node tool-filter.js",
            "description": "根據上下文過濾可用工具"
          }
        ]
      }
    ]
  }
}

// tool-filter.js 範例輸出
console.log(JSON.stringify({
  hookSpecificOutput: {
    hookEventName: 'BeforeToolSelection',
    toolConfig: {
      mode: 'ANY',
      allowedFunctionNames: ['read_file', 'run_shell_command']
    }
  }
}));
```

---

## 10. 安全考量

### 10.1 專案 Hook 信任機制

#### 10.1.1 信任管理器

```typescript
/**
 * TrustedHooksManager
 * 管理專案級 hooks 的信任狀態
 */
class TrustedHooksManager {
  private configPath: string;           // ~/.gemini/trusted_hooks.json
  private trustedHooks: TrustedHooksConfig = {};

  constructor() {
    this.configPath = path.join(
      Storage.getGlobalGeminiDir(),
      'trusted_hooks.json',
    );
    this.load();
  }

  /**
   * 獲取專案中未信任的 hooks
   */
  getUntrustedHooks(
    projectPath: string,
    hooks: { [K in HookEventName]?: HookDefinition[] },
  ): string[] {
    const trustedKeys = new Set(this.trustedHooks[projectPath] || []);
    const untrusted: string[] = [];

    for (const eventName of Object.keys(hooks)) {
      const definitions = hooks[eventName as HookEventName];
      if (!Array.isArray(definitions)) continue;

      for (const def of definitions) {
        for (const hook of def.hooks) {
          const key = getHookKey(hook);  // 格式: "name:command"
          if (!trustedKeys.has(key)) {
            untrusted.push(hook.name || hook.command || 'unknown-hook');
          }
        }
      }
    }

    return Array.from(new Set(untrusted));
  }

  /**
   * 信任專案中的所有 hooks
   */
  trustHooks(
    projectPath: string,
    hooks: { [K in HookEventName]?: HookDefinition[] },
  ): void {
    const currentTrusted = new Set(this.trustedHooks[projectPath] || []);

    for (const eventName of Object.keys(hooks)) {
      const definitions = hooks[eventName as HookEventName];
      for (const def of definitions) {
        for (const hook of def.hooks) {
          currentTrusted.add(getHookKey(hook));
        }
      }
    }

    this.trustedHooks[projectPath] = Array.from(currentTrusted);
    this.save();
  }
}
```

#### 10.1.2 信任檢查流程

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           專案 Hook 信任檢查流程                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

                           載入專案 Hooks 配置
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │ 資料夾是否已被信任?            │
                    │ this.config.isTrustedFolder() │
                    └───────────────┬───────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │ 否                            │ 是
                    ▼                               ▼
         ┌──────────────────────┐     ┌──────────────────────────────┐
         │ 警告: 專案 hooks 已   │     │ 檢查 hooks 是否已信任         │
         │ 禁用，因為資料夾未   │     │ checkProjectHooksTrust()     │
         │ 被信任               │     └───────────────┬──────────────┘
         └──────────────────────┘                     │
                                      ┌───────────────┴───────────────┐
                                      │ 有未信任的                    │ 全部已信任
                                      ▼                               ▼
                           ┌──────────────────────┐     ┌──────────────────────┐
                           │ 顯示警告訊息:        │     │ 正常載入 hooks       │
                           │                      │     └──────────────────────┘
                           │ "WARNING: 檢測到     │
                           │  以下專案級 hooks:   │
                           │  - hook1             │
                           │  - hook2             │
                           │                      │
                           │ 這些 hooks 將被執行"│
                           │                      │
                           │ 自動信任這些 hooks   │
                           │ trustHooks()         │
                           └──────────────────────┘
```

#### 10.1.3 執行時安全檢查

```typescript
// HookRunner 中的二次安全檢查
async executeHook(hookConfig, eventName, input): Promise<HookExecutionResult> {
  // 二次安全檢查：確保專案 hooks 不會在未信任的資料夾中執行
  if (
    hookConfig.source === ConfigSource.Project &&
    !this.config.isTrustedFolder()
  ) {
    const errorMessage = 'Security: Blocked execution of project hook in untrusted folder';
    debugLogger.warn(errorMessage);
    return {
      hookConfig,
      eventName,
      success: false,
      error: new Error(errorMessage),
      duration: 0,
    };
  }

  // 繼續正常執行...
}
```

### 10.2 環境變數淨化

#### 10.2.1 淨化配置

```typescript
type EnvironmentSanitizationConfig = {
  allowedEnvironmentVariables: string[];      // 明確允許的變數
  blockedEnvironmentVariables: string[];      // 明確禁止的變數
  enableEnvironmentVariableRedaction: boolean; // 是否啟用淨化
};
```

#### 10.2.2 永遠允許的變數

```typescript
const ALWAYS_ALLOWED_ENVIRONMENT_VARIABLES: ReadonlySet<string> = new Set([
  // 跨平台
  'PATH',

  // Windows 特定
  'SYSTEMROOT', 'COMSPEC', 'PATHEXT', 'WINDIR',
  'TEMP', 'TMP', 'USERPROFILE', 'SYSTEMDRIVE',

  // Unix/Linux/macOS 特定
  'HOME', 'LANG', 'SHELL', 'TMPDIR', 'USER', 'LOGNAME',

  // GitHub Action 相關
  'ADDITIONAL_CONTEXT', 'AVAILABLE_LABELS', 'BRANCH_NAME',
  'DESCRIPTION', 'EVENT_NAME', 'GITHUB_ENV', 'IS_PULL_REQUEST',
  // ... 等等
]);
```

#### 10.2.3 永遠禁止的變數

```typescript
const NEVER_ALLOWED_ENVIRONMENT_VARIABLES: ReadonlySet<string> = new Set([
  'CLIENT_ID',
  'DB_URI',
  'CONNECTION_STRING',
  'AWS_DEFAULT_REGION',
  'AZURE_CLIENT_ID',
  'AZURE_TENANT_ID',
  'SLACK_WEBHOOK_URL',
  'TWILIO_ACCOUNT_SID',
  'DATABASE_URL',
  'GOOGLE_CLOUD_PROJECT',
  'GOOGLE_CLOUD_ACCOUNT',
  'FIREBASE_PROJECT_ID',
]);
```

#### 10.2.4 敏感模式檢測

```typescript
// 變數名稱中的敏感模式
const NEVER_ALLOWED_NAME_PATTERNS = [
  /TOKEN/i,
  /SECRET/i,
  /PASSWORD/i,
  /PASSWD/i,
  /KEY/i,
  /AUTH/i,
  /CREDENTIAL/i,
  /CREDS/i,
  /PRIVATE/i,
  /CERT/i,
] as const;

// 變數值中的敏感模式
const NEVER_ALLOWED_VALUE_PATTERNS = [
  /-----BEGIN (RSA|OPENSSH|EC|PGP) PRIVATE KEY-----/i,
  /-----BEGIN CERTIFICATE-----/i,
  /(https?|ftp|smtp):\/\/[^:]+:[^@]+@/i,           // URL 中的憑證
  /(ghp|gho|ghu|ghs|ghr|github_pat)_[a-zA-Z0-9_]{36,}/i,  // GitHub tokens
  /AIzaSy[a-zA-Z0-9_\\-]{33}/i,                    // Google API keys
  /AKIA[A-Z0-9]{16}/i,                              // AWS Access Key ID
  /eyJ[a-zA-Z0-9_-]*\.[a-zA-Z0-9_-]*\.[a-zA-Z0-9_-]*/i,  // JWT tokens
  /(s|r)k_(live|test)_[0-9a-zA-Z]{24}/i,           // Stripe API keys
  /xox[abpr]-[a-zA-Z0-9-]+/i,                      // Slack tokens
] as const;
```

#### 10.2.5 淨化邏輯

```typescript
function sanitizeEnvironment(
  processEnv: NodeJS.ProcessEnv,
  config: EnvironmentSanitizationConfig,
): NodeJS.ProcessEnv {
  if (!config.enableEnvironmentVariableRedaction) {
    return { ...processEnv };
  }

  const results: NodeJS.ProcessEnv = {};
  const allowedSet = new Set(config.allowedEnvironmentVariables.map(k => k.toUpperCase()));
  const blockedSet = new Set(config.blockedEnvironmentVariables.map(k => k.toUpperCase()));

  // GitHub Actions 中啟用嚴格模式
  const isStrictSanitization = !!processEnv['GITHUB_SHA'];

  for (const key in processEnv) {
    const value = processEnv[key];
    if (!shouldRedactEnvironmentVariable(key, value, allowedSet, blockedSet, isStrictSanitization)) {
      results[key] = value;
    }
  }

  return results;
}

function shouldRedactEnvironmentVariable(
  key: string,
  value: string | undefined,
  allowedSet?: Set<string>,
  blockedSet?: Set<string>,
  isStrictSanitization = false,
): boolean {
  key = key.toUpperCase();

  // 使用者覆蓋優先
  if (allowedSet?.has(key)) return false;
  if (blockedSet?.has(key)) return true;

  // 永遠允許的變數
  if (ALWAYS_ALLOWED_ENVIRONMENT_VARIABLES.has(key) || key.startsWith('GEMINI_CLI_')) {
    return false;
  }

  // 永遠禁止的變數
  if (NEVER_ALLOWED_ENVIRONMENT_VARIABLES.has(key)) return true;

  // 嚴格模式下，未明確允許的都會被過濾
  if (isStrictSanitization) return true;

  // 檢查名稱模式
  for (const pattern of NEVER_ALLOWED_NAME_PATTERNS) {
    if (pattern.test(key)) return true;
  }

  // 檢查值模式
  if (value) {
    for (const pattern of NEVER_ALLOWED_VALUE_PATTERNS) {
      if (pattern.test(value)) return true;
    }
  }

  return false;
}
```

### 10.3 程序生命週期管理

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              程序生命週期管理                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

                              Hook 程序啟動
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  1. 程序生成配置                                                                  │
│                                                                                  │
│  spawn(executable, args, {                                                       │
│    env: sanitizedEnv,        // 已淨化的環境變數                                  │
│    cwd: input.cwd,           // 工作目錄                                         │
│    stdio: ['pipe', 'pipe', 'pipe'],  // stdin, stdout, stderr 都是 pipe         │
│    shell: false              // 不使用 shell 執行 (防止注入)                      │
│  })                                                                              │
└──────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  2. 超時計時器設定                                                                │
│                                                                                  │
│  const timeoutHandle = setTimeout(() => {                                        │
│    timedOut = true;                                                              │
│    child.kill('SIGTERM');         // 優雅終止                                    │
│                                                                                  │
│    setTimeout(() => {                                                            │
│      if (!child.killed) {                                                        │
│        child.kill('SIGKILL');     // 強制終止 (5秒後)                            │
│      }                                                                           │
│    }, 5000);                                                                     │
│  }, timeout);                     // 預設 60000ms                                │
└──────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  3. 輸入發送 (stdin)                                                              │
│                                                                                  │
│  try {                                                                           │
│    child.stdin.write(JSON.stringify(input));                                     │
│    child.stdin.end();                                                            │
│  } catch (err) {                                                                 │
│    // 忽略 EPIPE 錯誤 (程序提前關閉 stdin)                                        │
│    if (err.code !== 'EPIPE') {                                                   │
│      debugLogger.debug(`Hook stdin error: ${err}`);                              │
│    }                                                                             │
│  }                                                                               │
└──────────────────────────────────────────────────────────────────────────────────┘
                                    │
                              ┌─────┴─────┐
                              │           │
                              ▼           ▼
                       ┌──────────┐ ┌──────────┐
                       │ 收集     │ │ 收集     │
                       │ stdout   │ │ stderr   │
                       └────┬─────┘ └────┬─────┘
                            │            │
                            └─────┬──────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  4. 程序結束處理                                                                  │
│                                                                                  │
│  child.on('close', (exitCode) => {                                               │
│    clearTimeout(timeoutHandle);       // 清除超時計時器                           │
│    const duration = Date.now() - startTime;                                      │
│                                                                                  │
│    if (timedOut) {                                                               │
│      // 超時情況                                                                  │
│      resolve({                                                                   │
│        success: false,                                                           │
│        error: new Error(`Hook timed out after ${timeout}ms`),                    │
│        stdout, stderr, duration                                                  │
│      });                                                                         │
│    } else {                                                                      │
│      // 正常結束                                                                  │
│      // 解析輸出並返回結果                                                        │
│    }                                                                             │
│  });                                                                             │
│                                                                                  │
│  child.on('error', (error) => {                                                  │
│    clearTimeout(timeoutHandle);                                                  │
│    resolve({                                                                     │
│      success: false,                                                             │
│      error,                                                                      │
│      stdout, stderr, duration                                                    │
│    });                                                                           │
│  });                                                                             │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 10.4 命令變數展開與注入防護

```typescript
/**
 * 展開命令中的變數，使用適當的 shell 轉義
 */
private expandCommand(command: string, input: HookInput, shellType: ShellType): string {
  debugLogger.debug(`Expanding hook command: ${command} (cwd: ${input.cwd})`);

  // 使用 shell 類型適當的轉義函式
  const escapedCwd = escapeShellArg(input.cwd, shellType);

  return command
    .replace(/\$GEMINI_PROJECT_DIR/g, () => escapedCwd)
    .replace(/\$CLAUDE_PROJECT_DIR/g, () => escapedCwd);
}

/**
 * 根據 shell 類型轉義參數
 */
function escapeShellArg(arg: string, shell: ShellType): string {
  if (!arg) return '';

  switch (shell) {
    case 'powershell':
      // PowerShell: 用單引號包裹，內部單引號加倍
      return `'${arg.replace(/'/g, "''")}'`;

    case 'cmd':
      // cmd.exe: 用雙引號包裹，內部雙引號加倍
      return `"${arg.replace(/"/g, '""')}"`;

    case 'bash':
    default:
      // POSIX shell: 使用 shell-quote 函式庫
      return quote([arg]);
  }
}
```

---

## 11. 關鍵設計模式

### 11.1 策略模式: 聚合策略

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         策略模式: 聚合策略選擇                                   │
└─────────────────────────────────────────────────────────────────────────────────┘

                              HookAggregator
                                    │
                                    │ aggregateResults(results, eventName)
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
         ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────┐
         │ OrDecisionStrategy│ │FieldReplacement│ │ToolSelectionStrategy│
         │                 │ │ Strategy        │ │                     │
         │ 適用事件:        │ │ 適用事件:       │ │ 適用事件:           │
         │ - BeforeTool    │ │ - BeforeModel   │ │ - BeforeToolSelection│
         │ - AfterTool     │ │ - AfterModel    │ │                     │
         │ - BeforeAgent   │ │                 │ │ 特性:               │
         │ - AfterAgent    │ │ 特性:           │ │ - 工具名稱 UNION    │
         │ - SessionStart  │ │ - 後覆蓋前      │ │ - 模式取最嚴格      │
         │                 │ │ - 增量合併      │ │ - 排序確保確定性    │
         │ 特性:           │ │                 │ │                     │
         │ - 任何阻擋獲勝   │ │                 │ │                     │
         │ - 訊息連接       │ │                 │ │                     │
         │ - continue 傳播  │ │                 │ │                     │
         └─────────────────┘ └─────────────────┘ └─────────────────────┘
```

### 11.2 轉換器模式: SDK 解耦

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          轉換器模式: SDK 版本解耦                                │
└─────────────────────────────────────────────────────────────────────────────────┘

                      抽象基類: HookTranslator
                      ┌─────────────────────────────────┐
                      │ abstract toHookLLMRequest()     │
                      │ abstract fromHookLLMRequest()   │
                      │ abstract toHookLLMResponse()    │
                      │ abstract fromHookLLMResponse()  │
                      │ abstract toHookToolConfig()     │
                      │ abstract fromHookToolConfig()   │
                      └──────────────┬──────────────────┘
                                     │
                                     │ 繼承
                                     │
                      ┌──────────────▼──────────────────┐
                      │ HookTranslatorGenAIv1           │
                      │                                 │
                      │ 當前實作:                        │
                      │ - 處理 GenAI SDK v1.x 類型      │
                      │ - 僅提取文字內容                 │
                      │ - 過濾非文字部分                 │
                      └─────────────────────────────────┘
                                     │
                                     │ 未來擴展
                                     ▼
                      ┌─────────────────────────────────┐
                      │ HookTranslatorGenAIv2 (未來)    │
                      │                                 │
                      │ 可能的擴展:                      │
                      │ - 支援多模態內容                 │
                      │ - 新的 API 結構                 │
                      │ - 函數呼叫的詳細轉換            │
                      └─────────────────────────────────┘

優點:
- Hook 腳本不受 SDK 版本變更影響
- 可以針對不同 SDK 版本創建不同的轉換器
- 穩定的 Hook API 合約
```

### 11.3 註冊表模式: Hook 發現與管理

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           註冊表模式: Hook 管理                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

                              HookRegistry
                      ┌─────────────────────────────────┐
                      │                                 │
                      │  職責:                          │
                      │  - 多來源配置載入               │
                      │  - Hook 驗證                   │
                      │  - 來源追蹤                    │
                      │  - 啟用/禁用管理               │
                      │  - 優先順序排序                │
                      │                                 │
                      │  內部結構:                      │
                      │  entries: HookRegistryEntry[]  │
                      │                                 │
                      └──────────────┬──────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
          ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
          │ Project Source  │ │ User Source     │ │ Extension Source│
          │                 │ │                 │ │                 │
          │ .gemini/        │ │ ~/.gemini/      │ │ MCP 擴展        │
          │ settings.json   │ │ settings.json   │ │ hooks 配置      │
          │                 │ │                 │ │                 │
          │ 需要信任檢查    │ │ 無需額外檢查    │ │ 需要啟用狀態    │
          └─────────────────┘ └─────────────────┘ └─────────────────┘

使用方式:

  // 獲取特定事件的 hooks (已排序、已過濾)
  const hooks = registry.getHooksForEvent(HookEventName.BeforeTool);

  // 獲取所有 hooks (用於管理介面)
  const allHooks = registry.getAllHooks();

  // 啟用/禁用特定 hook
  registry.setHookEnabled('my-security-hook', false);
```

### 11.4 觀察者模式: 事件驅動執行

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         觀察者模式: 事件驅動 Hook 執行                           │
└─────────────────────────────────────────────────────────────────────────────────┘

                              事件發布者
                    ┌─────────────────────────────────┐
                    │ Agent / LLMClient / ToolExecutor│
                    │                                 │
                    │ 觸發事件:                        │
                    │ - BeforeTool                    │
                    │ - AfterModel                    │
                    │ - SessionStart                  │
                    │ - ...                           │
                    └──────────────┬──────────────────┘
                                   │
                                   │ 觸發
                                   ▼
                    ┌─────────────────────────────────┐
                    │       HookEventHandler          │
                    │                                 │
                    │ fire*Event() 方法:              │
                    │ - fireBeforeToolEvent()         │
                    │ - fireAfterModelEvent()         │
                    │ - fireSessionStartEvent()       │
                    │ - ...                           │
                    └──────────────┬──────────────────┘
                                   │
                                   │ 通知所有匹配的觀察者
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
    ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │   Hook A        │  │   Hook B        │  │   Hook C        │
    │   (安全檢查)    │  │   (稽核日誌)    │  │   (內容過濾)    │
    │                 │  │                 │  │                 │
    │ matcher: *      │  │ matcher: write* │  │ matcher: read*  │
    └─────────────────┘  └─────────────────┘  └─────────────────┘

特點:
- 事件發布者與觀察者 (hooks) 解耦
- 新增 hook 不需要修改事件發布者
- matcher 支援選擇性訂閱
- 支援並行和順序執行
```

### 11.5 工廠模式: Hook 輸出類別創建

```typescript
/**
 * 工廠函式: 根據事件類型創建適當的輸出類別
 */
function createHookOutput(eventName: string, data: Partial<HookOutput>): DefaultHookOutput {
  switch (eventName) {
    case 'BeforeModel':
      return new BeforeModelHookOutput(data);    // 支援合成回應
    case 'AfterModel':
      return new AfterModelHookOutput(data);     // 支援回應修改
    case 'BeforeToolSelection':
      return new BeforeToolSelectionHookOutput(data);  // 支援工具配置
    case 'BeforeTool':
      return new BeforeToolHookOutput(data);     // 支援輸入修改
    default:
      return new DefaultHookOutput(data);        // 基本功能
  }
}

// 使用場景: HookAggregator 中
private createSpecificHookOutput(output: HookOutput, eventName: HookEventName): DefaultHookOutput {
  switch (eventName) {
    case HookEventName.BeforeTool:
      return new BeforeToolHookOutput(output);
    case HookEventName.BeforeModel:
      return new BeforeModelHookOutput(output);
    // ...
  }
}
```

---

## 總結

Gemini CLI Hooks 系統是一個精心設計的可擴展架構，具有以下關鍵特性：

### 架構優勢

| 特性 | 說明 |
|------|------|
| **模組化設計** | 6 個獨立元件，職責明確分離，易於維護和測試 |
| **版本穩定性** | HookTranslator 模式隔離 SDK 變更，確保 hook 腳本的長期相容性 |
| **彈性執行** | 支援並行和順序執行，針對不同事件類型有不同的聚合策略 |
| **安全性** | 多層信任系統、環境變數淨化、程序生命週期管理 |
| **可觀察性** | 完整的遙測日誌、調試輸出、錯誤報告機制 |
| **可擴展性** | 易於添加新的 hook 事件類型或聚合策略 |

### 事件覆蓋範圍

系統提供 11 種 hook 事件，覆蓋了 CLI 操作的各個關鍵點：

- **工具層**: BeforeTool, AfterTool, BeforeToolSelection
- **代理層**: BeforeAgent, AfterAgent
- **模型層**: BeforeModel, AfterModel
- **會話層**: SessionStart, SessionEnd
- **其他**: Notification, PreCompress

### 安全機制

- **專案 Hook 信任**: 需要使用者明確信任專案級 hooks
- **環境淨化**: 自動過濾敏感環境變數和憑證
- **程序隔離**: 透過 stdin/stdout JSON 通訊，避免直接 shell 注入
- **超時管理**: SIGTERM + SIGKILL 雙階段清理機制

### 設計模式應用

- **策略模式**: 事件特定的聚合策略
- **轉換器模式**: SDK 版本解耦
- **註冊表模式**: 集中化的 hook 管理
- **觀察者模式**: 事件驅動的 hook 執行
- **工廠模式**: 輸出類別的創建

這個 Hooks 系統為 Gemini CLI 提供了強大的可定制性，同時保持了良好的安全性和穩定性。
