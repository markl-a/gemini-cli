# Hooks 系統深度分析報告

## 系統架構概覽

### 高階架構圖

```
┌─────────────────────────────────────────────────────────────────┐
│                        HookSystem                               │
│  協調所有 hook 相關元件的主要協調器                              │
└───────────┬──────────────────────────────────────────────┬───────┘
            │                                              │
    ┌───────▼──────────┐                         ┌────────▼────────┐
    │  HookRegistry    │                         │  HookEventHandler│
    │  (發現)          │                         │  (事件觸發)      │
    └────────┬─────────┘                         └────────┬────────┘
             │                                           │
    ┌────────▼──────────┐                        ┌──────▼──────────┐
    │   HookPlanner     │◄──────────────────────►│   HookRunner    │
    │  (匹配 & 規劃)    │                        │   (執行)        │
    └────────┬──────────┘                        └──────┬──────────┘
             │                                         │
             └──────────────┬──────────────────────────┘
                           │
                    ┌──────▼──────────┐
                    │ HookAggregator  │
                    │ (合併結果)       │
                    └──────┬──────────┘
                           │
                    ┌──────▼──────────┐
                    │ HookTranslator  │
                    │ (格式轉換)       │
                    └─────────────────┘
```

### 元件職責

| 元件 | 職責 | 關鍵方法 |
|------|------|----------|
| **HookRegistry** | 從配置載入、驗證和發現 hooks | `initialize()`, `getHooksForEvent()`, `setHookEnabled()` |
| **HookPlanner** | 匹配 hooks 到上下文並創建執行計劃 | `createExecutionPlan()`, `matchesContext()` |
| **HookRunner** | 在 shell 中執行命令 hooks | `executeHook()`, `executeHooksParallel()`, `executeHooksSequential()` |
| **HookAggregator** | 使用事件特定策略合併多個 hooks 的結果 | `aggregateResults()`, `mergeOutputs()` |
| **HookTranslator** | 在 SDK 類型和 hook 穩定格式之間轉換 | `toHookLLMRequest()`, `fromHookLLMResponse()` |
| **HookEventHandler** | 協調所有元件並觸發事件 | `fireBeforeToolEvent()`, `fireAfterModelEvent()` 等 |

---

## Hook 註冊表

### 配置來源 (優先順序)
```
專案 > 使用者 > 系統 > 擴展
  (1)     (2)     (3)      (4)
```
較高優先順序的來源會覆蓋較低的。

### 關鍵資料結構

```typescript
// 帶完整元資料的註冊表條目
interface HookRegistryEntry {
  config: HookConfig;           // 實際 hook 配置
  source: ConfigSource;         // 來源 (Project/User/System/Extensions)
  eventName: HookEventName;     // 監聽的事件
  matcher?: string;             // 模式匹配器 (工具名或觸發器)
  sequential?: boolean;         // 執行模式指示
  enabled: boolean;             // 啟用/禁用標記
}

// Hook 命令配置
interface CommandHookConfig {
  type: HookType.Command;
  command: string;              // 要執行的 Shell 命令
  name?: string;                // 可選友好名稱
  description?: string;         // 可選描述
  timeout?: number;             // 最大執行時間 (ms)
  source?: ConfigSource;        // 註冊時填充
}
```

---

## Hook 類型與事件系統

### 支援的 Hook 事件 (11 種)

```typescript
enum HookEventName {
  BeforeTool = 'BeforeTool',              // 工具執行前
  AfterTool = 'AfterTool',                // 工具執行後
  BeforeAgent = 'BeforeAgent',            // 代理提示處理前
  Notification = 'Notification',          // 通知事件
  AfterAgent = 'AfterAgent',              // 代理回應後
  SessionStart = 'SessionStart',          // 會話初始化
  SessionEnd = 'SessionEnd',              // 會話終止
  PreCompress = 'PreCompress',            // 歷史壓縮前
  BeforeModel = 'BeforeModel',            // LLM API 呼叫前
  AfterModel = 'AfterModel',              // LLM API 呼叫後
  BeforeToolSelection = 'BeforeToolSelection', // 工具選擇邏輯前
}
```

### Hook 輸入/輸出架構

#### 基礎結構

```typescript
// 所有 hook 事件通用
interface HookInput {
  session_id: string;           // 當前會話 ID
  transcript_path: string;      // 對話歷史路徑
  cwd: string;                  // 當前工作目錄
  hook_event_name: string;      // 觸發的事件名稱
  timestamp: string;            // ISO 時間戳
}

// 大多數 hook 事件通用
interface HookOutput {
  continue?: boolean;           // 繼續執行 (false = 停止)
  stopReason?: string;          // 停止原因
  suppressOutput?: boolean;     // 從記錄中隱藏
  systemMessage?: string;       // 顯示給使用者
  decision?: HookDecision;      // 'ask' | 'block' | 'deny' | 'approve' | 'allow'
  reason?: string;              // 決策原因
  hookSpecificOutput?: Record<string, unknown>; // 事件特定資料
}
```

---

## Hook 執行器

### 執行模式

```typescript
// 並行執行 (預設)
async executeHooksParallel(
  hookConfigs: HookConfig[],
  eventName: HookEventName,
  input: HookInput
): Promise<HookExecutionResult[]>

// 順序執行 (如果任何 hook 要求)
async executeHooksSequential(
  hookConfigs: HookConfig[],
  eventName: HookEventName,
  input: HookInput
): Promise<HookExecutionResult[]>
```

### 單一 Hook 執行流程

```
輸入驗證
    │
    ├─► 檢查信任資料夾 (僅專案 hooks)
    │   │
    │   └─► 如果不信任則拒絕
    │
    ├─► 展開命令變數
    │   ├─ $GEMINI_PROJECT_DIR → input.cwd
    │   └─ $CLAUDE_PROJECT_DIR → input.cwd (相容性)
    │
    ├─► 生成 shell 程序
    │   ├─ 執行檔: bash/cmd/pwsh
    │   ├─ 工作目錄: input.cwd
    │   └─ 環境: 已淨化 + GEMINI_PROJECT_DIR
    │
    ├─► 將輸入作為 JSON 發送到 stdin
    │
    ├─► 收集 stdout/stderr
    │
    ├─► 等待退出 + 超時處理
    │   │
    │   └─► 如果超時: SIGTERM 然後 SIGKILL
    │
    ├─► 解析輸出
    │   ├─ exitCode == 0: 將 stdout 解析為 JSON
    │   ├─ exitCode == 2: 阻擋錯誤 (deny)
    │   └─ exitCode == 1: 非阻擋錯誤 (warning)
    │
    └─► 返回 HookExecutionResult
```

### 退出碼語義

| 退出碼 | 含義 | Hook 決策 |
|--------|------|-----------|
| 0 | 成功 | 'allow' (或自訂 JSON) |
| 1 | 非阻擋錯誤 | 'allow' + 警告訊息 |
| 2 | 阻擋錯誤 | 'deny' + 錯誤原因 |
| 其他 | 非阻擋錯誤 | 'allow' + 警告訊息 |

---

## Hook 聚合器

### 聚合策略選擇

```typescript
// OR 邏輯 (任何 hook 可以阻擋)
case HookEventName.BeforeTool:
case HookEventName.AfterTool:
case HookEventName.BeforeAgent:
  → mergeWithOrDecision()

// 欄位替換 (後面的 hooks 覆蓋前面的)
case HookEventName.BeforeModel:
case HookEventName.AfterModel:
  → mergeWithFieldReplacement()

// 特殊: 工具選擇 (工具的 UNION)
case HookEventName.BeforeToolSelection:
  → mergeToolSelectionOutputs()
```

### OR 決策邏輯

```
決策合併:
Output 1: decision='allow'   → 阻擋? 否
Output 2: decision='deny'    → 阻擋? 是
─────────────────────────────────────────
Result:  decision='deny'      → 被阻擋

邏輯: 任何阻擋決策獲勝

Continue 標記合併:
Output 1: continue=true
Output 2: continue=false     → 停止執行
─────────────────────────────────────────
Result:  continue=false      → 停止執行

邏輯: continue=false 傳播 (任何 hook 可以停止)
```

---

## Hook 轉換器

### 版本相容性問題

```
挑戰: Hook API 必須在 SDK 版本間保持穩定
解決方案: 轉換層提供抽象

SDK Version 1.x ──┐
                 ├──► HookTranslator ──► 穩定 Hook API
SDK Version 2.x ──┘
                 (未來)
```

### 解耦格式規範

**LLMRequest (穩定 Hook 格式)**
```typescript
interface LLMRequest {
  model: string;
  messages: Array<{
    role: 'user' | 'model' | 'system';
    content: string | Array<{ type: string; [key: string]: unknown }>;
  }>;
  config?: {
    temperature?: number;
    maxOutputTokens?: number;
    topP?: number;
    // ...更多
  };
  toolConfig?: HookToolConfig;
}
```

---

## 執行流程序列

### 序列 1: BeforeTool Hook 執行

```
┌─ 使用者呼叫工具 ─┐
│                    │
│ fireBeforeToolEvent(toolName, toolInput)
│ │
│ ├─ 創建帶 tool_name 和 tool_input 的輸入
│ │
│ ├─ createExecutionPlan(BeforeTool, {toolName})
│ │  │
│ │  ├─ 從註冊表獲取所有 BeforeTool hooks
│ │  │
│ │  ├─ 按 matcher 過濾 (正規表達式匹配工具名)
│ │  │
│ │  ├─ 去重 hooks
│ │  │
│ │  └─ 判斷: 順序或並行?
│ │
│ ├─ 執行 hooks (並行/順序)
│ │
│ ├─ aggregateResults(results, BeforeTool)
│ │  │
│ │  └─ mergeWithOrDecision()
│ │
│ └─ 返回 AggregatedHookResult
│
├─ 如果被阻擋: 阻止工具執行，顯示原因
│
└─ 否則: 繼續工具執行
```

---

## 整合點

### 1. 工具執行整合

```
工具執行流程:
    │
    ├─► fireBeforeToolEvent(toolName, input)
    │   │
    │   └─ Hook 可以:
    │      ├─ 修改工具輸入
    │      └─ 阻擋工具執行 (decision=deny)
    │
    ├─► 執行工具
    │
    └─► fireAfterToolEvent(toolName, input, response)
        │
        └─ Hook 可以:
           └─ 添加 additionalContext 到輸出
```

### 2. 模型執行整合

```
模型呼叫流程:
    │
    ├─► fireBeforeModelEvent(llmRequest)
    │   │
    │   └─ Hook 可以:
    │      ├─ 修改 llm_request
    │      ├─ 返回合成 llm_response (跳過 API 呼叫)
    │      └─ 阻擋執行 (continue=false)
    │
    ├─► 呼叫模型 (如果未被阻擋)
    │
    └─► fireAfterModelEvent(llmRequest, llmResponse)
        │
        └─ Hook 可以:
           ├─ 修改 llm_response
           └─ 完全覆蓋回應
```

---

## 安全考量

### 1. 專案 Hook 信任
- 專案 hooks 僅在信任的資料夾中執行
- 不信任的 hooks 觸發警告
- 使用者必須明確信任 hooks
- 信任的 hooks 列表儲存在 `~/.gemini/trusted_hooks.json`

### 2. 環境變數淨化
- 執行 hook 前移除敏感環境變數
- 僅傳遞 `GEMINI_PROJECT_DIR` 和 `CLAUDE_PROJECT_DIR`
- 使用服務的 `sanitizeEnvironment()`

### 3. 程序生命週期管理
- 超時強制執行 (預設 60 秒)
- SIGTERM → SIGKILL 清理
- 透過 JSON 的 stdin/stdout 隔離
- 透過適當轉義無 shell 注入

---

## 關鍵設計模式

### 1. 策略模式: 聚合
不同事件類型使用不同聚合策略:
- **OR 邏輯**: BeforeTool, AfterTool, BeforeAgent, AfterAgent
- **欄位替換**: BeforeModel, AfterModel
- **特殊**: BeforeToolSelection (工具模式優先順序)

### 2. 轉換器模式: SDK 解耦
HookTranslator 抽象類別與實現:
- 將 SDK 類型轉換為穩定 hook 格式
- 可以交換版本特定的轉換器
- 目前: HookTranslatorGenAIv1

### 3. 註冊表模式: Hook 發現
HookRegistry 集中化:
- Hook 配置載入
- 驗證
- 來源追蹤
- 啟用/禁用管理

---

## 總結

Gemini CLI Hooks 系統提供複雜、可擴展的機制，用於在多個點攔截和修改執行:

### 關鍵成就:
1. **模組化設計**: 6 個獨立元件，職責明確
2. **版本穩定性**: 轉換器模式隔離 SDK 變更
3. **彈性執行**: 並行/順序與事件特定合併
4. **安全性**: 信任系統、環境淨化、超時管理
5. **可觀察性**: 遙測日誌、調試輸出、錯誤報告
6. **可擴展性**: 易於添加新的 hook 事件或聚合策略
