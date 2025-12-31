# Core Package 深度分析報告

## 1. API 客戶端層

### BaseLlmClient (`/packages/core/src/core/baseLlmClient.ts`)
**大小**: 340 行 | **類別**: `BaseLlmClient` (Lines 101-339)

#### 用途
無狀態、工具導向的 LLM 呼叫，用於 JSON 生成、嵌入和內容生成，帶有自動重試邏輯和回退處理。

#### 關鍵方法

```typescript
// Line 108-159: 生成結構化 JSON 回應
async generateJson(options: GenerateJsonOptions): Promise<Record<string, unknown>>

// Line 161-194: 透過 API 生成嵌入
async generateEmbedding(texts: string[]): Promise<number[][]>

// Line 209-238: 生成內容回應
async generateContent(options: GenerateContentOptions): Promise<GenerateContentResponse>

// Line 240-338: 核心重試邏輯 (private)
private async _generateWithRetry(...)
```

#### 重試策略
```typescript
// 重試策略:
// 1. 應用模型選擇和可用性政策
// 2. 調用 retryWithBackoff():
//    - shouldRetryOnContent 回調 (line 300)
//    - maxAttempts (預設 5, line 31)
//    - getAvailabilityContext 提供者 (line 265-268)
//    - onPersistent429 回調用於回退 (line 304-306)
```

---

## 2. 聊天系統 - Turn 管理與對話流程

### GeminiChat (`/packages/core/src/core/geminiChat.ts`)
**大小**: 28,575 bytes | **類別**: `GeminiChat` (Lines 207-881)

#### 用途
核心聊天會話管理，維護對話歷史、處理串流回應、管理重試與無效內容偵測，以及協調 hooks。

#### 類別建構子與狀態 (Lines 207-227)
```typescript
constructor(
  private readonly config: Config,
  private systemInstruction: string = '',
  private tools: Tool[] = [],
  private history: Content[] = [],
  resumedSessionData?: ResumedSessionData
)

// 狀態:
// - sendPromise: 序列化訊息發送以防止並發呼叫
// - chatRecordingService: 記錄所有訊息、工具呼叫、tokens
// - lastPromptTokenCount: 追蹤用於遙測
```

#### 主要訊息串流 (Lines 258-379)

```typescript
async sendMessageStream(
  modelConfigKey: ModelConfigKey,
  message: PartListUnion,
  prompt_id: string,
  signal: AbortSignal
): Promise<AsyncGenerator<StreamEvent>>
```

**流程** (Lines 292-378):
1. 等待上一個訊息發送完成 (line 264)
2. 創建使用者內容並記錄訊息 (lines 272-286)
3. 將使用者內容加入歷史 (line 289)
4. 返回處理重試的生成器 (lines 292-378)

#### 串流回應處理 (Lines 699-817)

```typescript
private async *processStreamResponse(
  model: string,
  streamResponse: AsyncGenerator<GenerateContentResponse>,
  originalRequest: GenerateContentParameters
): AsyncGenerator<GenerateContentResponse>
```

**關鍵邏輯:**
1. 收集模型回應部分 (lines 704-731)
2. 追蹤工具呼叫和思考 (lines 724-730)
3. 記錄 token 使用量 (lines 734-740)
4. 為每個 chunk 觸發 AfterModel hook (lines 746-751)
5. 合併文字部分 (lines 758-770)
6. 驗證串流完成 (lines 787-814)

---

## 3. 工具調度器 - 工具執行協調

### CoreToolScheduler (`/packages/core/src/core/coreToolScheduler.ts`)
**大小**: 39,650 bytes | **類別**: `CoreToolScheduler` (Lines 114-1179)

#### 用途
協調工具執行生命週期：驗證、確認、執行、錯誤處理和結果收集。實現帶有順序處理和批次完成的佇列。

#### 工具呼叫狀態

```typescript
type ToolCall =
  | ValidatingToolCall      // 檢查確認需求
  | ScheduledToolCall       // 已批准，等待執行
  | ExecutingToolCall       // 正在執行
  | SuccessfulToolCall      // 成功完成
  | ErroredToolCall         // 執行期間失敗
  | CancelledToolCall       // 使用者取消
  | WaitingToolCall         // 等待使用者批准
```

#### 主要調度流程 (Lines 410-450)

```typescript
async schedule(
  request: ToolCallRequestInfo | ToolCallRequestInfo[],
  signal: AbortSignal
): Promise<void>

// 邏輯:
// 1. 如果已在執行/調度中: 加入佇列並等待
// 2. 否則: 直接呼叫 _schedule()
```

#### 執行 (Lines 831-1034)

```typescript
private async attemptExecutionOfScheduledCalls(
  signal: AbortSignal
): Promise<void>

// 對每個已調度的工具呼叫:
// 1. 設置狀態 → executing
// 2. 創建輸出回調 (如果工具支援即時更新)
// 3. 處理特殊 ShellToolInvocation
// 4. 使用 hooks 執行
// 5. 成功時: 可選截斷輸出，轉換為函數回應
// 6. 錯誤時: 設置狀態 → error
// 7. 檢查批次完成
```

---

## 4. 提示建構 - 系統提示與上下文

### Prompts (`/packages/core/src/core/prompts.ts`)
**大小**: 34,079 bytes

#### 用途
建構注入每個 API 請求的綜合系統提示，包括核心指令、工具定義、專案結構上下文、使用者記憶和壓縮提示。

#### 核心系統提示 (Lines 80-200)

```typescript
export function getCoreSystemPrompt(
  config: Config,
  userMemory?: string
): string

// 組裝:
// 1. 檢查 GEMINI_SYSTEM_MD 環境變數是否有自訂覆蓋
// 2. 從 ~/.gemini/system.md 載入預設 system.md
// 3. 建構額外上下文區段:
//    a. 包含使用者記憶 (如果提供)
//    b. 專案目錄上下文
//    c. Git 儲存庫上下文 (如果適用)
//    d. 支援的工具列表
// 4. 返回串接的提示
```

---

## 5. 會話管理

### ChatRecordingService (`/packages/core/src/services/chatRecordingService.ts`)
**大小**: 495 行 | **類別**: `ChatRecordingService` (Lines 111-495)

#### 用途
自動將所有對話資料記錄到磁碟以進行持久化、恢復和審計追蹤。將會話儲存為 JSON 檔案，包含完整的訊息歷史、工具呼叫、tokens 和思考。

#### 會話儲存結構 (Lines 83-98)

```typescript
export interface ConversationRecord {
  sessionId: string;
  projectHash: string;
  startTime: string;
  lastUpdated: string;
  messages: MessageRecord[];
  summary?: string;
}
```

#### 檔案儲存
```
會話檔案儲存在:
~/.gemini/tmp/<PROJECT_HASH>/chats/session-<TIMESTAMP>-<SESSION_ID>.json
```

---

## 6. 串流 - 回應處理與即時輸出

### 串流事件類型 (Lines 58-68)

```typescript
export enum StreamEventType {
  CHUNK = 'chunk',    // 常規 API 回應 chunk
  RETRY = 'retry'     // 重試信號 (丟棄先前 chunks)
}
```

### 生成器模式 (Lines 258-379)

```typescript
async sendMessageStream(...): Promise<AsyncGenerator<StreamEvent>>

// 返回非同步生成器:
// 1. 處理多次重試嘗試
// 2. 在每次重試時產出 RETRY 事件
// 3. 為每個回應產出 CHUNK 事件
// 4. 管理退避延遲: 500ms * (attempt + 1)
```

---

## 序列圖

### 1. 帶重試的訊息發送流程

```
使用者請求
    |
    v
sendMessageStream()
    |
    +---> 等待 sendPromise (序列化)
    |
    +---> 記錄使用者訊息
    |
    +---> 附加到歷史
    |
    +---> 創建 streamWithRetries 生成器
            |
            +---> For attempt = 0 to maxAttempts-1
                    |
                    +---> [如果 attempt > 0] yield RETRY
                    |
                    +---> makeApiCallAndProcessStream()
                    |
                    +---> processStreamResponse()
                    |
                    +---> [On InvalidStreamError]
                            |
                            +---> 如果可重試: 日誌、睡眠、繼續
                            +---> 否則: 中斷並拋出
    |
    +---> 解決 streamDonePromise
    |
    v
返回生成器
```

### 2. 工具執行協調

```
schedule(request)
    |
    +---> [如果已在執行] 加入佇列並返回
    |
    +---> _schedule(request)
            |
            +---> 創建 ToolCall 物件
            +---> 加入 toolCallQueue
            +---> _processNextInQueue()
                    |
                    +---> 從佇列移出第一個 → toolCalls[0]
                    +---> 驗證確認需求
                    +---> 執行已調度的呼叫
                    +---> checkAndNotifyCompletion()
    |
    v
所有工具完成時返回
```

---

## 錯誤處理模式

### 1. 無效串流偵測

**位置**: `geminiChat.ts:699-817` (processStreamResponse)

```typescript
// 驗證條件:
if (!hasToolCall) {
  if (!finishReason)
    throw new InvalidStreamError('No finish reason', 'NO_FINISH_REASON')

  if (finishReason === FinishReason.MALFORMED_FUNCTION_CALL)
    throw new InvalidStreamError('Malformed call', 'MALFORMED_FUNCTION_CALL')

  if (!responseText)
    throw new InvalidStreamError('Empty response', 'NO_RESPONSE_TEXT')
}
```

### 2. 工具執行錯誤處理

**位置**: `coreToolScheduler.ts:831-1028`

```typescript
try {
  const toolResult = await executeToolWithHooks(...)
  // 成功路徑
} catch (executionError) {
  if (signal.aborted) {
    setStatusInternal(callId, 'cancelled', signal, ...)
  } else {
    setStatusInternal(callId, 'error', signal, createErrorResponse(...))
  }
}
```

---

## 關鍵架構模式

### 1. 串流的非同步生成器模式
- `sendMessageStream()` 返回 `AsyncGenerator<StreamEvent>`
- 啟用即時回應處理和重試邏輯
- RETRY 事件允許 UI 清除部分內容

### 2. 工具執行的狀態機
- 帶有嚴格狀態轉換的順序工具處理
- 狀態: validating → scheduled → executing → (success|error|cancelled)
- 透過佇列架構防止並發執行

### 3. 歷史策展
- 綜合歷史: 所有 turns 包括無效的 (用於調試)
- 策展歷史: 只有有效 turns 發送到 API (extractCuratedHistory)
- 防止累積無效回應導致的 API 錯誤

### 4. 帶指數退避的重試
- 網路錯誤: 帶可配置最大嘗試次數的 retryWithBackoff
- 內容錯誤: 僅對 Gemini 2 特殊重試 (InvalidStreamError)
- 速率限制: 持續 429 時回退到替代模型

### 5. 基於 Hook 的擴展點
- BeforeModel: 在 API 呼叫前修改內容/配置
- BeforeToolSelection: 修改工具定義
- AfterModel: 後處理回應 chunks
- 工具特定的執行生命週期 hooks

---

## 關鍵行號參考

| 元件 | 檔案 | 關鍵行 |
|------|------|--------|
| GeminiChat 類別 | geminiChat.ts | 207-227, 258-379, 381-549, 699-817 |
| sendMessageStream | geminiChat.ts | 258-379 |
| 無效串流偵測 | geminiChat.ts | 90-127, 184-198, 787-814 |
| Turn 事件生成 | turn.ts | 222-351 |
| CoreToolScheduler | coreToolScheduler.ts | 114-1179 |
| 工具調度 | coreToolScheduler.ts | 410-450, 483-554 |
| 工具執行 | coreToolScheduler.ts | 831-1034 |
| 會話記錄 | chatRecordingService.ts | 111-495 |
