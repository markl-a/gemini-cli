# Core Package 深度架構分析報告

**版本**: 2.0 | **更新日期**: 2025
**套件位置**: `/packages/core/src/core/`

---

## 目錄

1. [架構概觀](#1-架構概觀)
2. [API 客戶端層](#2-api-客戶端層)
3. [聊天系統](#3-聊天系統)
4. [工具調度器](#4-工具調度器)
5. [提示建構](#5-提示建構)
6. [會話管理](#6-會話管理)
7. [串流處理](#7-串流處理)
8. [錯誤處理模式](#8-錯誤處理模式)
9. [關鍵架構模式](#9-關鍵架構模式)
10. [序列圖](#10-序列圖)
11. [關鍵行號參考](#11-關鍵行號參考)

---

## 1. 架構概觀

Core 套件是 Gemini CLI 的核心引擎，負責管理與 Gemini API 的所有互動。它採用分層架構設計，將關注點分離為多個協作的子系統。

### 1.1 核心元件關係圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                           使用者介面層                               │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Turn (回合管理器)                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐ │
│  │ 事件生成        │  │ 串流消費        │  │ 錯誤轉換            │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌──────────────────────┐  ┌─────────────────┐  ┌──────────────────────┐
│      GeminiChat      │  │ CoreToolScheduler│  │ ChatRecordingService │
│  ┌────────────────┐  │  │  ┌───────────┐  │  │  ┌────────────────┐  │
│  │ 歷史管理       │  │  │  │ 狀態機    │  │  │  │ 持久化        │  │
│  │ 串流處理       │  │  │  │ 佇列管理  │  │  │  │ 記錄工具呼叫  │  │
│  │ 重試邏輯       │  │  │  │ 確認流程  │  │  │  │ Token 追蹤    │  │
│  └────────────────┘  │  │  └───────────┘  │  │  └────────────────┘  │
└──────────────────────┘  └─────────────────┘  └──────────────────────┘
           │                       │
           ▼                       ▼
┌──────────────────────┐  ┌─────────────────────────────────────────────┐
│     BaseLlmClient    │  │           Hook 系統 (擴展點)                 │
│  ┌────────────────┐  │  │  ┌─────────────┐  ┌──────────────────────┐ │
│  │ JSON 生成      │  │  │  │ BeforeModel │  │ BeforeToolSelection  │ │
│  │ 嵌入生成       │  │  │  └─────────────┘  └──────────────────────┘ │
│  │ 重試策略       │  │  │  ┌─────────────┐  ┌──────────────────────┐ │
│  └────────────────┘  │  │  │ AfterModel  │  │ BeforeTool/AfterTool │ │
└──────────────────────┘  │  └─────────────┘  └──────────────────────┘ │
           │               └─────────────────────────────────────────────┘
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     ContentGenerator (API 包裝層)                    │
│                         ↓                                            │
│                   @google/genai SDK                                  │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心檔案清單

| 檔案 | 大小 | 主要職責 |
|------|------|----------|
| `geminiChat.ts` | 891 行 | 聊天會話管理、串流處理、歷史管理 |
| `coreToolScheduler.ts` | 1,180 行 | 工具執行生命週期、狀態機、佇列管理 |
| `baseLlmClient.ts` | 339 行 | 無狀態 LLM 呼叫、JSON/嵌入生成 |
| `prompts.ts` | 452 行 | 系統提示建構、上下文組裝 |
| `turn.ts` | 393 行 | 回合管理、事件生成 |
| `chatRecordingService.ts` | 495 行 | 會話持久化、記錄管理 |

---

## 2. API 客戶端層

### 2.1 BaseLlmClient 詳細分析

**檔案**: `/packages/core/src/core/baseLlmClient.ts`
**類別**: `BaseLlmClient` (Lines 101-339)

BaseLlmClient 是一個專注於無狀態、工具導向 LLM 呼叫的客戶端類別。它提供 JSON 生成、嵌入向量生成和一般內容生成功能，並內建完整的重試機制。

#### 2.1.1 類別結構

```typescript
// Lines 101-106
export class BaseLlmClient {
  constructor(
    private readonly contentGenerator: ContentGenerator,  // API 呼叫代理
    private readonly config: Config,                       // 全域配置
    private readonly authType?: AuthType,                  // 認證類型
  ) {}
```

#### 2.1.2 核心方法詳解

**generateJson() - 結構化 JSON 回應生成**

```typescript
// Lines 108-159
async generateJson(
  options: GenerateJsonOptions,
): Promise<Record<string, unknown>> {
  const {
    schema,           // 必要的 JSON Schema
    modelConfigKey,   // 模型配置鍵
    contents,         // 輸入內容
    systemInstruction,// 系統指令
    abortSignal,      // 取消信號
    promptId,         // 遙測用的提示 ID
    maxAttempts,      // 最大嘗試次數
  } = options;

  // 內容驗證回調：檢查回應是否為有效 JSON
  const shouldRetryOnContent = (response: GenerateContentResponse) => {
    const text = getResponseText(response)?.trim();
    if (!text) return true;  // 空回應需要重試

    try {
      JSON.parse(this.cleanJsonResponse(text, model));
      return false;  // JSON 有效，不需重試
    } catch (_e) {
      return true;   // JSON 無效，需要重試
    }
  };

  // 使用重試機制生成內容
  const result = await this._generateWithRetry(
    { ...options, additionalProperties: {
      responseJsonSchema: schema,
      responseMimeType: 'application/json',
    }},
    shouldRetryOnContent,
    'generateJson',
  );

  // 清理並解析 JSON 回應
  return JSON.parse(
    this.cleanJsonResponse(getResponseText(result)!.trim(), model),
  );
}
```

**generateEmbedding() - 嵌入向量生成**

```typescript
// Lines 161-194
async generateEmbedding(texts: string[]): Promise<number[][]> {
  if (!texts || texts.length === 0) return [];

  const embedModelParams: EmbedContentParameters = {
    model: this.config.getEmbeddingModel(),
    contents: texts,
  };

  const embedContentResponse =
    await this.contentGenerator.embedContent(embedModelParams);

  // 驗證回應完整性
  if (!embedContentResponse.embeddings ||
      embedContentResponse.embeddings.length === 0) {
    throw new Error('No embeddings found in API response.');
  }

  if (embedContentResponse.embeddings.length !== texts.length) {
    throw new Error(
      `API returned a mismatched number of embeddings. ` +
      `Expected ${texts.length}, got ${embedContentResponse.embeddings.length}.`
    );
  }

  // 提取並驗證每個嵌入向量
  return embedContentResponse.embeddings.map((embedding, index) => {
    const values = embedding.values;
    if (!values || values.length === 0) {
      throw new Error(
        `API returned an empty embedding for input text at index ${index}`
      );
    }
    return values;
  });
}
```

### 2.2 重試策略深度分析

**檔案**: `/packages/core/src/utils/retry.ts`

#### 2.2.1 重試配置結構

```typescript
// Lines 21-35
export interface RetryOptions {
  maxAttempts: number;           // 最大嘗試次數
  initialDelayMs: number;        // 初始延遲 (毫秒)
  maxDelayMs: number;            // 最大延遲上限
  shouldRetryOnError: (error: Error, retryFetchErrors?: boolean) => boolean;
  shouldRetryOnContent?: (content: GenerateContentResponse) => boolean;
  onPersistent429?: (authType?: string, error?: unknown) => Promise<string | boolean | null>;
  authType?: string;
  retryFetchErrors?: boolean;
  signal?: AbortSignal;
  getAvailabilityContext?: () => RetryAvailabilityContext | undefined;
}

// 預設配置 (Lines 37-42)
const DEFAULT_RETRY_OPTIONS: RetryOptions = {
  maxAttempts: 3,
  initialDelayMs: 5000,      // 5 秒
  maxDelayMs: 30000,         // 30 秒
  shouldRetryOnError: isRetryableError,
};
```

#### 2.2.2 可重試錯誤判定

```typescript
// Lines 85-116
export function isRetryableError(
  error: Error | unknown,
  retryFetchErrors?: boolean,
): boolean {
  // 1. 檢查網路錯誤代碼
  const errorCode = getNetworkErrorCode(error);
  if (errorCode && RETRYABLE_NETWORK_CODES.includes(errorCode)) {
    return true;  // ECONNRESET, ETIMEDOUT, EPIPE, ENOTFOUND 等
  }

  // 2. 檢查 fetch 失敗
  if (retryFetchErrors && error instanceof Error) {
    if (error.message.toLowerCase().includes('fetch failed')) {
      return true;
    }
  }

  // 3. 檢查 API 錯誤狀態碼
  if (error instanceof ApiError) {
    if (error.status === 400) return false;  // 400 不重試
    return error.status === 429 ||           // 速率限制
           (error.status >= 500 && error.status < 600);  // 伺服器錯誤
  }

  // 4. 通用狀態碼檢查
  const status = getErrorStatus(error);
  if (status !== undefined) {
    return status === 429 || (status >= 500 && status < 600);
  }

  return false;
}
```

#### 2.2.3 指數退避演算法

```typescript
// Lines 125-290 (retryWithBackoff 核心邏輯)
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options?: Partial<RetryOptions>,
): Promise<T> {
  let attempt = 0;
  let currentDelay = initialDelayMs;

  while (attempt < maxAttempts) {
    if (signal?.aborted) throw createAbortError();
    attempt++;

    try {
      const result = await fn();

      // 內容驗證重試
      if (shouldRetryOnContent && shouldRetryOnContent(result)) {
        // 添加抖動的延遲
        const jitter = currentDelay * 0.3 * (Math.random() * 2 - 1);
        const delayWithJitter = Math.max(0, currentDelay + jitter);
        await delay(delayWithJitter, signal);
        currentDelay = Math.min(maxDelayMs, currentDelay * 2);  // 指數增長
        continue;
      }

      // 成功：標記模型健康
      const successContext = getAvailabilityContext?.();
      if (successContext) {
        successContext.service.markHealthy(successContext.policy.model);
      }
      return result;

    } catch (error) {
      // 錯誤分類與處理...
      const classifiedError = classifyGoogleError(error);

      // 終端配額錯誤或模型未找到：嘗試回退
      if (classifiedError instanceof TerminalQuotaError ||
          classifiedError instanceof ModelNotFoundError) {
        if (onPersistent429) {
          const fallbackModel = await onPersistent429(authType, classifiedError);
          if (fallbackModel) {
            attempt = 0;  // 重置嘗試次數
            currentDelay = initialDelayMs;
            continue;
          }
        }
        throw classifiedError;
      }

      // 可重試配額錯誤或 5xx 錯誤
      if (classifiedError instanceof RetryableQuotaError || is500) {
        if (attempt >= maxAttempts) {
          // 嘗試回退
          if (onPersistent429) {
            const fallbackModel = await onPersistent429(authType, classifiedError);
            if (fallbackModel) {
              attempt = 0;
              currentDelay = initialDelayMs;
              continue;
            }
          }
          throw classifiedError;
        }

        // 使用 API 提供的重試延遲或指數退避
        if (classifiedError.retryDelayMs !== undefined) {
          await delay(classifiedError.retryDelayMs, signal);
        } else {
          const jitter = currentDelay * 0.3 * (Math.random() * 2 - 1);
          await delay(Math.max(0, currentDelay + jitter), signal);
          currentDelay = Math.min(maxDelayMs, currentDelay * 2);
        }
        continue;
      }
    }
  }

  throw new Error('Retry attempts exhausted');
}
```

### 2.3 回退機制 (Fallback Handler)

**檔案**: `/packages/core/src/fallback/handler.ts`

```typescript
// Lines 23-112
export async function handleFallback(
  config: Config,
  failedModel: string,
  authType?: string,
  error?: unknown,
): Promise<string | boolean | null> {
  // 僅對 Google 登入認證啟用回退
  if (authType !== AuthType.LOGIN_WITH_GOOGLE) {
    return null;
  }

  // 1. 解析策略鏈
  const chain = resolvePolicyChain(config);
  const { failedPolicy, candidates } = buildFallbackPolicyContext(chain, failedModel);

  // 2. 分類錯誤類型
  const failureKind = classifyFailureKind(error);
  const availability = config.getModelAvailabilityService();

  // 3. 選擇可用的回退模型
  let fallbackModel: string;
  if (!candidates.length) {
    fallbackModel = failedModel;
  } else {
    const selection = availability.selectFirstAvailable(
      candidates.map((policy) => policy.model)
    );

    // 找到最後手段策略
    const lastResortPolicy = candidates.find((policy) => policy.isLastResort);
    const selectedFallbackModel = selection.selectedModel ?? lastResortPolicy?.model;

    if (!selectedFallbackModel || selectedFallbackModel === failedModel) {
      return null;
    }

    fallbackModel = selectedFallbackModel;

    // 4. 根據策略決定動作
    const action = resolvePolicyAction(failureKind, selectedPolicy);

    // 靜默回退：直接切換
    if (action === 'silent') {
      applyAvailabilityTransition(getAvailabilityContext, failureKind);
      return processIntent(config, 'retry_always', fallbackModel);
    }
  }

  // 5. 調用使用者提供的回退處理器
  const handler = config.getFallbackModelHandler();
  if (typeof handler !== 'function') return null;

  const intent = await handler(failedModel, fallbackModel, error);

  // 6. 處理使用者意圖
  if (intent === 'retry_always' || intent === 'retry_once') {
    applyAvailabilityTransition(getAvailabilityContext, failureKind);
  }

  return await processIntent(config, intent, fallbackModel);
}
```

#### 回退意圖處理

```typescript
// Lines 125-159
async function processIntent(
  config: Config,
  intent: FallbackIntent | null,
  fallbackModel: string,
): Promise<boolean> {
  switch (intent) {
    case 'retry_always':
      // 永久切換到回退模型
      config.setActiveModel(fallbackModel);
      return true;

    case 'retry_once':
      // 僅本次使用回退模型，不永久切換
      return true;

    case 'stop':
      // 停止執行，保持當前模型
      return false;

    case 'retry_later':
      // 稍後重試
      return false;

    case 'upgrade':
      // 開啟升級頁面
      await handleUpgrade();
      return false;

    default:
      throw new Error(`Unexpected fallback intent: "${intent}"`);
  }
}
```

---

## 3. 聊天系統

### 3.1 GeminiChat 完整分析

**檔案**: `/packages/core/src/core/geminiChat.ts`
**類別**: `GeminiChat` (Lines 207-881)

GeminiChat 是核心聊天會話管理器，負責維護對話歷史、處理串流回應、管理重試機制以及協調 Hooks。

#### 3.1.1 類別狀態與建構

```typescript
// Lines 207-227
export class GeminiChat {
  // 序列化訊息發送的 Promise
  private sendPromise: Promise<void> = Promise.resolve();

  // 會話記錄服務
  private readonly chatRecordingService: ChatRecordingService;

  // 最後一次提示的 token 計數
  private lastPromptTokenCount: number;

  constructor(
    private readonly config: Config,           // 全域配置
    private systemInstruction: string = '',    // 系統指令
    private tools: Tool[] = [],                // 可用工具
    private history: Content[] = [],           // 對話歷史
    resumedSessionData?: ResumedSessionData,   // 恢復會話資料
  ) {
    validateHistory(history);  // 驗證歷史格式
    this.chatRecordingService = new ChatRecordingService(config);
    this.chatRecordingService.initialize(resumedSessionData);

    // 初始化 token 計數
    this.lastPromptTokenCount = estimateTokenCountSync(
      this.history.flatMap((c) => c.parts || []),
    );
  }
```

#### 3.1.2 訊息串流發送 (核心方法)

```typescript
// Lines 258-379
async sendMessageStream(
  modelConfigKey: ModelConfigKey,
  message: PartListUnion,
  prompt_id: string,
  signal: AbortSignal,
): Promise<AsyncGenerator<StreamEvent>> {
  // 1. 等待前一個訊息完成 (序列化)
  await this.sendPromise;

  // 2. 創建完成 Promise 用於序列化
  let streamDoneResolver: () => void;
  const streamDonePromise = new Promise<void>((resolve) => {
    streamDoneResolver = resolve;
  });
  this.sendPromise = streamDonePromise;

  // 3. 創建使用者內容並記錄
  const userContent = createUserContent(message);
  const { model } = this.config.modelConfigService.getResolvedConfig(modelConfigKey);

  // 記錄非工具回應的使用者訊息
  if (!isFunctionResponse(userContent)) {
    const userMessage = Array.isArray(message) ? message : [message];
    const userMessageContent = partListUnionToString(toParts(userMessage));
    this.chatRecordingService.recordMessage({
      model,
      type: 'user',
      content: userMessageContent,
    });
  }

  // 4. 將使用者內容加入歷史 (在任何嘗試前只加入一次)
  this.history.push(userContent);
  const requestContents = this.getHistory(true);  // 獲取策展後的歷史

  // 5. 創建帶重試的非同步生成器
  const streamWithRetries = async function* (
    this: GeminiChat,
  ): AsyncGenerator<StreamEvent, void, void> {
    try {
      let lastError: unknown = new Error('Request failed after all retries.');
      const maxAttempts = INVALID_CONTENT_RETRY_OPTIONS.maxAttempts;  // 2 次

      for (let attempt = 0; attempt < maxAttempts; attempt++) {
        let isConnectionPhase = true;
        try {
          // 重試時發出 RETRY 事件
          if (attempt > 0) {
            yield { type: StreamEventType.RETRY };
          }

          // 更新配置鍵以標記重試
          const currentConfigKey = attempt > 0
            ? { ...modelConfigKey, isRetry: true }
            : modelConfigKey;

          // 連接階段：建立串流
          isConnectionPhase = true;
          const stream = await this.makeApiCallAndProcessStream(
            currentConfigKey,
            requestContents,
            prompt_id,
            signal,
          );
          isConnectionPhase = false;

          // 產出每個 chunk
          for await (const chunk of stream) {
            yield { type: StreamEventType.CHUNK, value: chunk };
          }

          lastError = null;
          break;  // 成功，退出重試迴圈

        } catch (error) {
          // 連接階段錯誤直接拋出
          if (isConnectionPhase) throw error;

          lastError = error;
          const isContentError = error instanceof InvalidStreamError;
          const isRetryable = isRetryableError(error, this.config.getRetryFetchErrors());

          // 判斷是否可重試
          if ((isContentError && isGemini2Model(model)) ||
              (isRetryable && !signal.aborted)) {
            if (attempt < maxAttempts - 1) {
              const delayMs = INVALID_CONTENT_RETRY_OPTIONS.initialDelayMs;
              const retryType = isContentError ? error.type : 'NETWORK_ERROR';

              // 記錄重試遙測
              logContentRetry(this.config, new ContentRetryEvent(
                attempt, retryType, delayMs, model
              ));

              // 線性退避延遲
              await new Promise((res) =>
                setTimeout(res, delayMs * (attempt + 1))
              );
              continue;
            }
          }
          break;
        }
      }

      if (lastError) {
        // 記錄重試失敗
        if (lastError instanceof InvalidStreamError && isGemini2Model(model)) {
          logContentRetryFailure(this.config, new ContentRetryFailureEvent(
            maxAttempts, lastError.type, model
          ));
        }
        throw lastError;
      }
    } finally {
      streamDoneResolver!();  // 解鎖下一個訊息發送
    }
  };

  return streamWithRetries.call(this);
}
```

#### 3.1.3 API 呼叫與串流處理

```typescript
// Lines 381-549
private async makeApiCallAndProcessStream(
  modelConfigKey: ModelConfigKey,
  requestContents: Content[],
  prompt_id: string,
  abortSignal: AbortSignal,
): Promise<AsyncGenerator<GenerateContentResponse>> {
  // 確保預覽模型的思考簽名
  const contentsForPreviewModel =
    this.ensureActiveLoopHasThoughtSignatures(requestContents);

  // 應用模型選擇策略
  const {
    model: availabilityFinalModel,
    config: newAvailabilityConfig,
    maxAttempts: availabilityMaxAttempts,
  } = applyModelSelection(this.config, modelConfigKey);

  let lastModelToUse = availabilityFinalModel;
  let currentGenerateContentConfig = newAvailabilityConfig;

  // 創建可用性上下文提供者
  const getAvailabilityContext = createAvailabilityContextProvider(
    this.config,
    () => lastModelToUse,
  );

  const initialActiveModel = this.config.getActiveModel();

  // API 呼叫閉包
  const apiCall = async () => {
    let modelToUse = resolveModel(lastModelToUse, this.config.getPreviewFeatures());

    // 檢測回退導致的模型變更
    if (this.config.getActiveModel() !== initialActiveModel) {
      modelToUse = resolveModel(
        this.config.getActiveModel(),
        this.config.getPreviewFeatures()
      );
    }

    // 模型變更時重新獲取配置
    if (modelToUse !== lastModelToUse) {
      const { generateContentConfig: newConfig } =
        this.config.modelConfigService.getResolvedConfig({
          ...modelConfigKey,
          model: modelToUse,
        });
      currentGenerateContentConfig = newConfig;
    }

    lastModelToUse = modelToUse;
    const config: GenerateContentConfig = {
      ...currentGenerateContentConfig,
      systemInstruction: this.systemInstruction,
      tools: this.tools,
      abortSignal,
    };

    let contentsToUse = isPreviewModel(modelToUse)
      ? contentsForPreviewModel
      : requestContents;

    // === Hook 整合 ===
    const hooksEnabled = this.config.getEnableHooks();
    const messageBus = this.config.getMessageBus();

    if (hooksEnabled && messageBus) {
      // BeforeModel Hook
      const beforeModelResult = await fireBeforeModelHook(messageBus, {
        model: modelToUse,
        config,
        contents: contentsToUse,
      });

      // 檢查是否被阻擋
      if (beforeModelResult.blocked) {
        const syntheticResponse = beforeModelResult.syntheticResponse;
        if (syntheticResponse) {
          return (async function* () { yield syntheticResponse; })();
        }
        return (async function* () {})();
      }

      // 應用修改
      if (beforeModelResult.modifiedConfig) {
        Object.assign(config, beforeModelResult.modifiedConfig);
      }
      if (beforeModelResult.modifiedContents) {
        contentsToUse = beforeModelResult.modifiedContents as Content[];
      }

      // BeforeToolSelection Hook
      const toolSelectionResult = await fireBeforeToolSelectionHook(messageBus, {
        model: modelToUse,
        config,
        contents: contentsToUse,
      });

      if (toolSelectionResult.toolConfig) {
        config.toolConfig = toolSelectionResult.toolConfig;
      }
      if (toolSelectionResult.tools) {
        config.tools = toolSelectionResult.tools as Tool[];
      }
    }

    // 發送 API 請求
    return this.config.getContentGenerator().generateContentStream(
      { model: modelToUse, contents: contentsToUse, config },
      prompt_id,
    );
  };

  // 429 持續錯誤回調
  const onPersistent429Callback = async (authType?: string, error?: unknown) =>
    handleFallback(this.config, lastModelToUse, authType, error);

  // 使用重試機制執行 API 呼叫
  const streamResponse = await retryWithBackoff(apiCall, {
    onPersistent429: onPersistent429Callback,
    authType: this.config.getContentGeneratorConfig()?.authType,
    retryFetchErrors: this.config.getRetryFetchErrors(),
    signal: abortSignal,
    maxAttempts: availabilityMaxAttempts,
    getAvailabilityContext,
  });

  // 儲存原始請求用於 AfterModel Hook
  const originalRequest: GenerateContentParameters = {
    model: lastModelToUse,
    config: lastConfig,
    contents: lastContentsToUse,
  };

  return this.processStreamResponse(lastModelToUse, streamResponse, originalRequest);
}
```

#### 3.1.4 串流回應處理與驗證

```typescript
// Lines 699-817
private async *processStreamResponse(
  model: string,
  streamResponse: AsyncGenerator<GenerateContentResponse>,
  originalRequest: GenerateContentParameters,
): AsyncGenerator<GenerateContentResponse> {
  const modelResponseParts: Part[] = [];
  let hasToolCall = false;
  let finishReason: FinishReason | undefined;

  for await (const chunk of streamResponse) {
    // 提取完成原因
    const candidateWithReason = chunk?.candidates?.find(
      (candidate) => candidate.finishReason
    );
    if (candidateWithReason) {
      finishReason = candidateWithReason.finishReason as FinishReason;
    }

    // 處理有效回應
    if (isValidResponse(chunk)) {
      const content = chunk.candidates?.[0]?.content;
      if (content?.parts) {
        // 記錄思考
        if (content.parts.some((part) => part.thought)) {
          this.recordThoughtFromContent(content);
        }
        // 標記工具呼叫
        if (content.parts.some((part) => part.functionCall)) {
          hasToolCall = true;
        }
        // 收集非思考部分
        modelResponseParts.push(
          ...content.parts.filter((part) => !part.thought)
        );
      }
    }

    // 記錄 token 使用量
    if (chunk.usageMetadata) {
      this.chatRecordingService.recordMessageTokens(chunk.usageMetadata);
      if (chunk.usageMetadata.promptTokenCount !== undefined) {
        this.lastPromptTokenCount = chunk.usageMetadata.promptTokenCount;
      }
    }

    // AfterModel Hook
    const hooksEnabled = this.config.getEnableHooks();
    const messageBus = this.config.getMessageBus();
    if (hooksEnabled && messageBus && originalRequest && chunk) {
      const hookResult = await fireAfterModelHook(messageBus, originalRequest, chunk);
      yield hookResult.response;
    } else {
      yield chunk;  // 立即產出每個 chunk
    }
  }

  // === 串流驗證邏輯 ===
  // 合併文字部分
  const consolidatedParts: Part[] = [];
  for (const part of modelResponseParts) {
    const lastPart = consolidatedParts[consolidatedParts.length - 1];
    if (lastPart?.text && isValidNonThoughtTextPart(lastPart) &&
        isValidNonThoughtTextPart(part)) {
      lastPart.text += part.text;
    } else {
      consolidatedParts.push(part);
    }
  }

  const responseText = consolidatedParts
    .filter((part) => part.text)
    .map((part) => part.text)
    .join('')
    .trim();

  // 記錄模型回應
  if (responseText) {
    this.chatRecordingService.recordMessage({
      model,
      type: 'gemini',
      content: responseText,
    });
  }

  // === 驗證條件 ===
  // 成功條件: 有工具呼叫 OR (有完成原因 AND 非空回應)
  // 錯誤條件: 無工具呼叫 AND (無完成原因 OR 畸形函數呼叫 OR 空回應)
  if (!hasToolCall) {
    if (!finishReason) {
      throw new InvalidStreamError(
        'Model stream ended without a finish reason.',
        'NO_FINISH_REASON'
      );
    }
    if (finishReason === FinishReason.MALFORMED_FUNCTION_CALL) {
      throw new InvalidStreamError(
        'Model stream ended with malformed function call.',
        'MALFORMED_FUNCTION_CALL'
      );
    }
    if (!responseText) {
      throw new InvalidStreamError(
        'Model stream ended with empty response text.',
        'NO_RESPONSE_TEXT'
      );
    }
  }

  // 將合併後的回應加入歷史
  this.history.push({ role: 'model', parts: consolidatedParts });
}
```

### 3.2 歷史管理與策展

#### 3.2.1 歷史類型

```typescript
// 兩種歷史類型:
// 1. 綜合歷史 (Comprehensive): 包含所有 turns，包括無效的
// 2. 策展歷史 (Curated): 只有有效 turns，發送給 API

// Lines 574-581
getHistory(curated: boolean = false): Content[] {
  const history = curated
    ? extractCuratedHistory(this.history)
    : this.history;
  // 深拷貝防止外部修改
  return structuredClone(history);
}
```

#### 3.2.2 歷史策展演算法

```typescript
// Lines 151-178
function extractCuratedHistory(comprehensiveHistory: Content[]): Content[] {
  if (!comprehensiveHistory || comprehensiveHistory.length === 0) {
    return [];
  }

  const curatedHistory: Content[] = [];
  const length = comprehensiveHistory.length;
  let i = 0;

  while (i < length) {
    if (comprehensiveHistory[i].role === 'user') {
      // 使用者訊息總是保留
      curatedHistory.push(comprehensiveHistory[i]);
      i++;
    } else {
      // 收集連續的模型輸出
      const modelOutput: Content[] = [];
      let isValid = true;

      while (i < length && comprehensiveHistory[i].role === 'model') {
        modelOutput.push(comprehensiveHistory[i]);
        if (isValid && !isValidContent(comprehensiveHistory[i])) {
          isValid = false;
        }
        i++;
      }

      // 只有當所有模型輸出都有效時才保留
      if (isValid) {
        curatedHistory.push(...modelOutput);
      }
    }
  }

  return curatedHistory;
}
```

### 3.3 Turn 回合管理

**檔案**: `/packages/core/src/core/turn.ts`
**類別**: `Turn` (Lines 210-392)

Turn 類別管理單個對話回合，負責將底層串流事件轉換為高階業務事件。

#### 3.3.1 事件類型定義

```typescript
// Lines 52-69
export enum GeminiEventType {
  Content = 'content',                    // 文字內容
  ToolCallRequest = 'tool_call_request',  // 工具呼叫請求
  ToolCallResponse = 'tool_call_response',// 工具呼叫回應
  ToolCallConfirmation = 'tool_call_confirmation', // 確認請求
  UserCancelled = 'user_cancelled',       // 使用者取消
  Error = 'error',                        // 錯誤
  ChatCompressed = 'chat_compressed',     // 聊天壓縮
  Thought = 'thought',                    // 思考
  MaxSessionTurns = 'max_session_turns',  // 達到最大回合數
  Finished = 'finished',                  // 完成
  LoopDetected = 'loop_detected',         // 偵測到迴圈
  Citation = 'citation',                  // 引用
  Retry = 'retry',                        // 重試
  ContextWindowWillOverflow = 'context_window_will_overflow', // 上下文將溢出
  InvalidStream = 'invalid_stream',       // 無效串流
  ModelInfo = 'model_info',               // 模型資訊
}
```

#### 3.3.2 Turn.run() 主流程

```typescript
// Lines 222-351
async *run(
  modelConfigKey: ModelConfigKey,
  req: PartListUnion,
  signal: AbortSignal,
): AsyncGenerator<ServerGeminiStreamEvent> {
  try {
    // 發送訊息並獲取串流
    const responseStream = await this.chat.sendMessageStream(
      modelConfigKey, req, this.prompt_id, signal
    );

    for await (const streamEvent of responseStream) {
      // 檢查取消
      if (signal?.aborted) {
        yield { type: GeminiEventType.UserCancelled };
        return;
      }

      // 處理 RETRY 事件
      if (streamEvent.type === 'retry') {
        yield { type: GeminiEventType.Retry };
        continue;
      }

      const resp = streamEvent.value;
      if (!resp) continue;

      this.debugResponses.push(resp);
      const traceId = resp.responseId;

      // 處理思考部分
      const thoughtPart = resp.candidates?.[0]?.content?.parts?.[0];
      if (thoughtPart?.thought) {
        const thought = parseThought(thoughtPart.text ?? '');
        yield { type: GeminiEventType.Thought, value: thought, traceId };
        continue;
      }

      // 處理文字內容
      const text = getResponseText(resp);
      if (text) {
        yield { type: GeminiEventType.Content, value: text, traceId };
      }

      // 處理函數呼叫
      const functionCalls = resp.functionCalls ?? [];
      for (const fnCall of functionCalls) {
        const event = this.handlePendingFunctionCall(fnCall, traceId);
        if (event) yield event;
      }

      // 收集引用
      for (const citation of getCitations(resp)) {
        this.pendingCitations.add(citation);
      }

      // 處理完成原因
      const finishReason = resp.candidates?.[0]?.finishReason;
      if (finishReason) {
        // 輸出引用
        if (this.pendingCitations.size > 0) {
          yield {
            type: GeminiEventType.Citation,
            value: `Citations:\n${[...this.pendingCitations].sort().join('\n')}`,
          };
          this.pendingCitations.clear();
        }

        this.finishReason = finishReason;
        yield {
          type: GeminiEventType.Finished,
          value: { reason: finishReason, usageMetadata: resp.usageMetadata },
        };
      }
    }
  } catch (e) {
    // 錯誤處理...
    if (signal.aborted) {
      yield { type: GeminiEventType.UserCancelled };
      return;
    }

    if (e instanceof InvalidStreamError) {
      yield { type: GeminiEventType.InvalidStream };
      return;
    }

    // 報告錯誤並產出錯誤事件
    const error = toFriendlyError(e);
    const structuredError: StructuredError = {
      message: getErrorMessage(error),
      status: error?.status,
    };
    await this.chat.maybeIncludeSchemaDepthContext(structuredError);
    yield { type: GeminiEventType.Error, value: { error: structuredError } };
  }
}
```

---

## 4. 工具調度器

### 4.1 CoreToolScheduler 完整分析

**檔案**: `/packages/core/src/core/coreToolScheduler.ts`
**類別**: `CoreToolScheduler` (Lines 114-1179)

CoreToolScheduler 是工具執行的核心協調器，實現了完整的工具生命週期管理，包括狀態機、佇列系統和確認流程。

### 4.2 工具呼叫狀態機

#### 4.2.1 狀態定義

```typescript
// /packages/core/src/scheduler/types.ts (Lines 38-115)
export type ValidatingToolCall = {
  status: 'validating';
  request: ToolCallRequestInfo;
  tool: AnyDeclarativeTool;
  invocation: AnyToolInvocation;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

export type ScheduledToolCall = {
  status: 'scheduled';
  request: ToolCallRequestInfo;
  tool: AnyDeclarativeTool;
  invocation: AnyToolInvocation;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

export type ExecutingToolCall = {
  status: 'executing';
  request: ToolCallRequestInfo;
  tool: AnyDeclarativeTool;
  invocation: AnyToolInvocation;
  liveOutput?: string | AnsiOutput;  // 即時輸出
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
  pid?: number;  // Shell 工具的進程 ID
};

export type WaitingToolCall = {
  status: 'awaiting_approval';
  request: ToolCallRequestInfo;
  tool: AnyDeclarativeTool;
  invocation: AnyToolInvocation;
  confirmationDetails: ToolCallConfirmationDetails;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

export type SuccessfulToolCall = {
  status: 'success';
  request: ToolCallRequestInfo;
  tool: AnyDeclarativeTool;
  response: ToolCallResponseInfo;
  invocation: AnyToolInvocation;
  durationMs?: number;
  outcome?: ToolConfirmationOutcome;
};

export type ErroredToolCall = {
  status: 'error';
  request: ToolCallRequestInfo;
  response: ToolCallResponseInfo;
  tool?: AnyDeclarativeTool;
  durationMs?: number;
  outcome?: ToolConfirmationOutcome;
};

export type CancelledToolCall = {
  status: 'cancelled';
  request: ToolCallRequestInfo;
  response: ToolCallResponseInfo;
  tool: AnyDeclarativeTool;
  invocation: AnyToolInvocation;
  durationMs?: number;
  outcome?: ToolConfirmationOutcome;
};
```

#### 4.2.2 狀態轉換圖

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                                                         │
                    ▼                                                         │
              ┌──────────┐                                                    │
              │validating│                                                    │
              └────┬─────┘                                                    │
                   │                                                          │
         ┌─────────┼─────────┐                                               │
         │         │         │                                               │
         ▼         ▼         ▼                                               │
    ┌────────┐ ┌────────┐ ┌─────────────────┐                               │
    │ error  │ │scheduled│ │awaiting_approval│                               │
    └────────┘ └────┬────┘ └────────┬────────┘                               │
                    │               │                                         │
                    │    ┌──────────┼──────────┐                             │
                    │    │          │          │                             │
                    ▼    ▼          ▼          ▼                             │
              ┌──────────┐    ┌─────────┐ ┌─────────┐                        │
              │ executing│    │scheduled│ │cancelled│─────────────────────►─┘
              └────┬─────┘    └────┬────┘ └─────────┘
                   │               │
         ┌─────────┼─────────┐     │
         │         │         │     │
         ▼         ▼         ▼     │
    ┌────────┐ ┌────────┐ ┌─────────┐
    │ error  │ │ success│ │cancelled│
    └────────┘ └────────┘ └─────────┘
         │         │           │
         └─────────┴───────────┘
                   │
                   ▼
           [終端狀態: 批次完成通知]
```

### 4.3 佇列管理系統

```typescript
// Lines 114-176
export class CoreToolScheduler {
  // 靜態 WeakMap 防止重複訂閱 MessageBus
  private static subscribedMessageBuses = new WeakMap<
    MessageBus,
    (request: ToolConfirmationRequest) => void
  >();

  private toolCalls: ToolCall[] = [];           // 當前活動的工具呼叫
  private outputUpdateHandler?: OutputUpdateHandler;
  private onAllToolCallsComplete?: AllToolCallsCompleteHandler;
  private onToolCallsUpdate?: ToolCallsUpdateHandler;
  private getPreferredEditor: () => EditorType | undefined;
  private config: Config;

  // 狀態標誌
  private isFinalizingToolCalls = false;        // 正在完成工具呼叫
  private isScheduling = false;                 // 正在調度
  private isCancelling = false;                 // 正在取消

  // 佇列系統
  private requestQueue: Array<{                 // 請求佇列
    request: ToolCallRequestInfo | ToolCallRequestInfo[];
    signal: AbortSignal;
    resolve: () => void;
    reject: (reason?: Error) => void;
  }> = [];
  private toolCallQueue: ToolCall[] = [];       // 工具呼叫佇列
  private completedToolCallsForBatch: CompletedToolCall[] = [];  // 批次完成列表
```

### 4.4 調度流程詳解

#### 4.4.1 schedule() 入口方法

```typescript
// Lines 410-450
schedule(
  request: ToolCallRequestInfo | ToolCallRequestInfo[],
  signal: AbortSignal,
): Promise<void> {
  return runInDevTraceSpan(
    { name: 'schedule' },
    async ({ metadata: spanMetadata }) => {
      spanMetadata.input = request;

      // 如果已在執行或調度中，加入佇列
      if (this.isRunning() || this.isScheduling) {
        return new Promise((resolve, reject) => {
          const abortHandler = () => {
            // 從佇列中移除已取消的請求
            const index = this.requestQueue.findIndex(
              (item) => item.request === request
            );
            if (index > -1) {
              this.requestQueue.splice(index, 1);
              reject(new Error('Tool call cancelled while in queue.'));
            }
          };

          signal.addEventListener('abort', abortHandler, { once: true });

          this.requestQueue.push({
            request,
            signal,
            resolve: () => {
              signal.removeEventListener('abort', abortHandler);
              resolve();
            },
            reject: (reason?: Error) => {
              signal.removeEventListener('abort', abortHandler);
              reject(reason);
            },
          });
        });
      }

      // 直接調度
      return this._schedule(request, signal);
    },
  );
}
```

#### 4.4.2 _schedule() 內部調度

```typescript
// Lines 483-554
private async _schedule(
  request: ToolCallRequestInfo | ToolCallRequestInfo[],
  signal: AbortSignal,
): Promise<void> {
  this.isScheduling = true;
  this.isCancelling = false;

  try {
    if (this.isRunning()) {
      throw new Error(
        'Cannot schedule new tool calls while other tool calls are actively running.'
      );
    }

    const requestsToProcess = Array.isArray(request) ? request : [request];
    this.completedToolCallsForBatch = [];

    // 為每個請求創建 ToolCall 物件
    const newToolCalls: ToolCall[] = requestsToProcess.map((reqInfo): ToolCall => {
      // 1. 查找工具實例
      const toolInstance = this.config.getToolRegistry().getTool(reqInfo.name);
      if (!toolInstance) {
        const suggestion = getToolSuggestion(
          reqInfo.name,
          this.config.getToolRegistry().getAllToolNames()
        );
        return {
          status: 'error',
          request: reqInfo,
          response: createErrorResponse(
            reqInfo,
            new Error(`Tool "${reqInfo.name}" not found.${suggestion}`),
            ToolErrorType.TOOL_NOT_REGISTERED
          ),
          durationMs: 0,
        };
      }

      // 2. 建構調用物件
      const invocationOrError = this.buildInvocation(toolInstance, reqInfo.args);
      if (invocationOrError instanceof Error) {
        return {
          status: 'error',
          request: reqInfo,
          tool: toolInstance,
          response: createErrorResponse(
            reqInfo,
            invocationOrError,
            ToolErrorType.INVALID_TOOL_PARAMS
          ),
          durationMs: 0,
        };
      }

      // 3. 創建驗證中狀態
      return {
        status: 'validating',
        request: reqInfo,
        tool: toolInstance,
        invocation: invocationOrError,
        startTime: Date.now(),
      };
    });

    // 加入工具呼叫佇列
    this.toolCallQueue.push(...newToolCalls);

    // 開始處理佇列
    await this._processNextInQueue(signal);
  } finally {
    this.isScheduling = false;
  }
}
```

#### 4.4.3 佇列處理流程

```typescript
// Lines 556-707
private async _processNextInQueue(signal: AbortSignal): Promise<void> {
  // 如果已有工具在處理或佇列為空，停止
  if (this.toolCalls.length > 0 || this.toolCallQueue.length === 0) {
    return;
  }

  // 處理取消
  if (signal.aborted) {
    this._cancelAllQueuedCalls();
    await this.checkAndNotifyCompletion(signal);
    return;
  }

  // 取出下一個工具呼叫
  const toolCall = this.toolCallQueue.shift()!;
  this.toolCalls = [toolCall];
  this.notifyToolCallsUpdate();

  // 處理已錯誤的工具
  if (toolCall.status === 'error') {
    await this.checkAndNotifyCompletion(signal);
    return;
  }

  // 處理驗證中的工具
  if (toolCall.status === 'validating') {
    const { request: reqInfo, invocation } = toolCall;

    try {
      if (signal.aborted) {
        this.setStatusInternal(reqInfo.callId, 'cancelled', signal,
          'Tool call cancelled by user.');
        await this.checkAndNotifyCompletion(signal);
        return;
      }

      // 檢查是否需要確認
      const confirmationDetails = await invocation.shouldConfirmExecute(signal);

      if (!confirmationDetails) {
        // 不需要確認，直接調度
        this.setToolCallOutcome(reqInfo.callId, ToolConfirmationOutcome.ProceedAlways);
        this.setStatusInternal(reqInfo.callId, 'scheduled', signal);
      } else {
        // 檢查自動批准
        if (this.isAutoApproved(toolCall)) {
          this.setToolCallOutcome(reqInfo.callId, ToolConfirmationOutcome.ProceedAlways);
          this.setStatusInternal(reqInfo.callId, 'scheduled', signal);
        } else {
          // 非互動模式不支持確認
          if (!this.config.isInteractive()) {
            throw new Error(
              `Tool "${toolCall.tool.displayName}" requires user confirmation, ` +
              `which is not supported in non-interactive mode.`
            );
          }

          // 觸發通知 Hook
          const messageBus = this.config.getMessageBus();
          const hooksEnabled = this.config.getEnableHooks();
          if (hooksEnabled && messageBus) {
            await fireToolNotificationHook(messageBus, confirmationDetails);
          }

          // 設置等待確認狀態
          const wrappedConfirmationDetails = {
            ...confirmationDetails,
            onConfirm: (outcome, payload) =>
              this.handleConfirmationResponse(
                reqInfo.callId, confirmationDetails.onConfirm,
                outcome, signal, payload
              ),
          };
          this.setStatusInternal(reqInfo.callId, 'awaiting_approval', signal,
            wrappedConfirmationDetails);
        }
      }
    } catch (error) {
      if (signal.aborted) {
        this.setStatusInternal(reqInfo.callId, 'cancelled', signal,
          'Tool call cancelled by user.');
      } else {
        this.setStatusInternal(reqInfo.callId, 'error', signal,
          createErrorResponse(reqInfo, error, ToolErrorType.UNHANDLED_EXCEPTION));
      }
      await this.checkAndNotifyCompletion(signal);
    }
  }

  await this.attemptExecutionOfScheduledCalls(signal);
}
```

### 4.5 工具執行流程

```typescript
// Lines 831-1034
private async attemptExecutionOfScheduledCalls(signal: AbortSignal): Promise<void> {
  // 檢查所有呼叫是否都在最終狀態或已調度
  const allCallsFinalOrScheduled = this.toolCalls.every(
    (call) => ['scheduled', 'cancelled', 'success', 'error'].includes(call.status)
  );

  if (allCallsFinalOrScheduled) {
    const callsToExecute = this.toolCalls.filter((call) => call.status === 'scheduled');

    for (const toolCall of callsToExecute) {
      if (toolCall.status !== 'scheduled') continue;

      const scheduledCall = toolCall;
      const { callId, name: toolName } = scheduledCall.request;
      const invocation = scheduledCall.invocation;

      // 設置為執行中
      this.setStatusInternal(callId, 'executing', signal);

      // 設置即時輸出回調
      const liveOutputCallback = scheduledCall.tool.canUpdateOutput && this.outputUpdateHandler
        ? (outputChunk: string | AnsiOutput) => {
            if (this.outputUpdateHandler) {
              this.outputUpdateHandler(callId, outputChunk);
            }
            this.toolCalls = this.toolCalls.map((tc) =>
              tc.request.callId === callId && tc.status === 'executing'
                ? { ...tc, liveOutput: outputChunk }
                : tc
            );
            this.notifyToolCallsUpdate();
          }
        : undefined;

      const shellExecutionConfig = this.config.getShellExecutionConfig();
      const hooksEnabled = this.config.getEnableHooks();
      const messageBus = this.config.getMessageBus();

      await runInDevTraceSpan({
        name: toolCall.tool.name,
        attributes: { type: 'tool-call' },
      }, async ({ metadata: spanMetadata }) => {
        spanMetadata.input = { request: toolCall.request };

        // Shell 工具特殊處理
        let promise: Promise<ToolResult>;
        if (invocation instanceof ShellToolInvocation) {
          const setPidCallback = (pid: number) => {
            this.toolCalls = this.toolCalls.map((tc) =>
              tc.request.callId === callId && tc.status === 'executing'
                ? { ...tc, pid }
                : tc
            );
            this.notifyToolCallsUpdate();
          };
          promise = executeToolWithHooks(
            invocation, toolName, signal, messageBus, hooksEnabled,
            toolCall.tool, liveOutputCallback, shellExecutionConfig, setPidCallback
          );
        } else {
          promise = executeToolWithHooks(
            invocation, toolName, signal, messageBus, hooksEnabled,
            toolCall.tool, liveOutputCallback, shellExecutionConfig
          );
        }

        try {
          const toolResult: ToolResult = await promise;
          spanMetadata.output = toolResult;

          if (signal.aborted) {
            this.setStatusInternal(callId, 'cancelled', signal,
              'User cancelled tool execution.');
          } else if (toolResult.error === undefined) {
            // 成功處理
            let content = toolResult.llmContent;
            let outputFile: string | undefined;

            // Shell 工具輸出截斷
            if (typeof content === 'string' && toolName === SHELL_TOOL_NAME &&
                this.config.getEnableToolOutputTruncation()) {
              const truncatedResult = await saveTruncatedContent(
                content, callId,
                this.config.storage.getProjectTempDir(),
                this.config.getTruncateToolOutputThreshold(),
                this.config.getTruncateToolOutputLines()
              );
              content = truncatedResult.content;
              outputFile = truncatedResult.outputFile;
            }

            const response = convertToFunctionResponse(
              toolName, callId, content, this.config.getActiveModel()
            );
            this.setStatusInternal(callId, 'success', signal, {
              callId,
              responseParts: response,
              resultDisplay: toolResult.returnDisplay,
              outputFile,
              contentLength: typeof content === 'string' ? content.length : undefined,
            });
          } else {
            // 錯誤處理
            this.setStatusInternal(callId, 'error', signal,
              createErrorResponse(scheduledCall.request,
                new Error(toolResult.error.message), toolResult.error.type));
          }
        } catch (executionError) {
          spanMetadata.error = executionError;
          if (signal.aborted) {
            this.setStatusInternal(callId, 'cancelled', signal,
              'User cancelled tool execution.');
          } else {
            this.setStatusInternal(callId, 'error', signal,
              createErrorResponse(scheduledCall.request,
                executionError instanceof Error ? executionError : new Error(String(executionError)),
                ToolErrorType.UNHANDLED_EXCEPTION));
          }
        }

        await this.checkAndNotifyCompletion(signal);
      });
    }
  }
}
```

### 4.6 確認回應處理

```typescript
// Lines 709-781
async handleConfirmationResponse(
  callId: string,
  originalOnConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>,
  outcome: ToolConfirmationOutcome,
  signal: AbortSignal,
  payload?: ToolConfirmationPayload,
): Promise<void> {
  const toolCall = this.toolCalls.find(
    (c) => c.request.callId === callId && c.status === 'awaiting_approval'
  );

  if (toolCall && toolCall.status === 'awaiting_approval') {
    await originalOnConfirm(outcome);
  }

  this.setToolCallOutcome(callId, outcome);

  // 處理不同的確認結果
  if (outcome === ToolConfirmationOutcome.Cancel || signal.aborted) {
    this.cancelAll(signal);
    return;
  } else if (outcome === ToolConfirmationOutcome.ModifyWithEditor) {
    // 使用編輯器修改
    const waitingToolCall = toolCall as WaitingToolCall;
    if (isModifiableDeclarativeTool(waitingToolCall.tool)) {
      // 設置修改中狀態
      this.setStatusInternal(callId, 'awaiting_approval', signal, {
        ...waitingToolCall.confirmationDetails,
        isModifying: true,
      });

      // 使用編輯器修改參數
      const { updatedParams, updatedDiff } = await modifyWithEditor(
        waitingToolCall.request.args,
        waitingToolCall.tool.getModifyContext(signal),
        this.getPreferredEditor()!,
        signal,
        /* contentOverrides */
      );

      this.setArgsInternal(callId, updatedParams);
      this.setStatusInternal(callId, 'awaiting_approval', signal, {
        ...waitingToolCall.confirmationDetails,
        fileDiff: updatedDiff,
        isModifying: false,
      });
    }
  } else {
    // 應用內聯修改（如果有）
    if (payload?.newContent && toolCall) {
      await this._applyInlineModify(toolCall as WaitingToolCall, payload, signal);
    }
    this.setStatusInternal(callId, 'scheduled', signal);
  }

  await this.attemptExecutionOfScheduledCalls(signal);
}
```

---

## 5. 提示建構

### 5.1 系統提示組裝流程

**檔案**: `/packages/core/src/core/prompts.ts`

#### 5.1.1 getCoreSystemPrompt() 主流程

```typescript
// Lines 80-386
export function getCoreSystemPrompt(config: Config, userMemory?: string): string {
  // 1. 檢查環境變數覆蓋
  let systemMdEnabled = false;
  let systemMdPath = path.resolve(path.join(GEMINI_DIR, 'system.md'));
  const systemMdResolution = resolvePathFromEnv(process.env['GEMINI_SYSTEM_MD']);

  if (systemMdResolution.value && !systemMdResolution.isDisabled) {
    systemMdEnabled = true;
    if (!systemMdResolution.isSwitch) {
      systemMdPath = systemMdResolution.value;
    }
    if (!fs.existsSync(systemMdPath)) {
      throw new Error(`missing system prompt file '${systemMdPath}'`);
    }
  }

  // 2. 解析模型配置
  const desiredModel = resolveModel(
    config.getActiveModel(),
    config.getPreviewFeatures()
  );
  const isGemini3 = isPreviewModel(desiredModel);
  const interactiveMode = config.isInteractiveShellEnabled();

  // 3. 建構提示配置
  let basePrompt: string;
  if (systemMdEnabled) {
    basePrompt = fs.readFileSync(systemMdPath, 'utf8');
  } else {
    const promptConfig = {
      preamble: `You are ${interactiveMode ? 'an interactive ' : 'a non-interactive '}` +
                `CLI agent specializing in software engineering tasks...`,

      coreMandates: `
# Core Mandates
- **Conventions:** Rigorously adhere to existing project conventions...
- **Libraries/Frameworks:** NEVER assume a library/framework is available...
- **Style & Structure:** Mimic the style, structure, framework choices...
- **Idiomatic Changes:** Understand the local context to ensure natural changes...
- **Comments:** Add code comments sparingly. Focus on *why*...
- **Proactiveness:** Fulfill the user's request thoroughly...
- **Do Not revert changes:** Do not revert changes unless asked...
`,

      primaryWorkflows_prefix: `
# Primary Workflows
## Software Engineering Tasks
1. **Understand:** Think about the user's request...
2. **Plan:** Build a coherent and grounded plan...
`,

      primaryWorkflows_suffix: `
3. **Implement:** Use the available tools...
4. **Verify (Tests):** If applicable, verify using tests...
5. **Verify (Standards):** Execute build, linting and type-checking...
6. **Finalize:** After all verification passes, consider complete...
`,

      operationalGuidelines: `
# Operational Guidelines
## Tone and Style
- **Concise & Direct:** Professional, direct tone...
- **Minimal Output:** Aim for fewer than 3 lines...
## Security and Safety Rules
- **Explain Critical Commands:** Before executing modifying commands...
- **Security First:** Never introduce code that exposes secrets...
## Tool Usage
- **Parallelism:** Execute multiple independent tool calls in parallel...
`,

      sandbox: `...`,  // 沙盒配置
      git: `...`,       // Git 配置
      finalReminder: `Your core function is efficient and safe assistance...`,
    };

    // 4. 組裝提示
    const orderedPrompts = ['preamble', 'coreMandates', ...];
    const enabledPrompts = orderedPrompts.filter((key) => {
      const envVar = process.env[`GEMINI_PROMPT_${key.toUpperCase()}`];
      return envVar !== '0' && envVar !== 'false';
    });

    basePrompt = enabledPrompts.map((key) => promptConfig[key]).join('\n');
  }

  // 5. 添加使用者記憶
  const memorySuffix = userMemory?.trim()
    ? `\n\n---\n\n${userMemory.trim()}`
    : '';

  return `${basePrompt}${memorySuffix}`;
}
```

### 5.2 提示區段結構

| 區段名稱 | 用途 | 可配置 |
|---------|------|--------|
| `preamble` | 角色定義和基本介紹 | 是 |
| `coreMandates` | 核心行為準則 | 是 |
| `primaryWorkflows_prefix` | 工作流程前半部分 | 是 |
| `primaryWorkflows_suffix` | 工作流程後半部分 | 是 |
| `operationalGuidelines` | 操作指南 | 是 |
| `sandbox` | 沙盒環境說明 | 是 |
| `git` | Git 操作指南 | 是 |
| `finalReminder` | 最終提醒 | 是 |

### 5.3 壓縮提示

```typescript
// Lines 393-451
export function getCompressionPrompt(): string {
  return `
You are the component that summarizes internal chat history into a given structure.

When the conversation history grows too large, you will be invoked to distill
the entire history into a concise, structured XML snapshot.

First, you will think through the entire history in a private <scratchpad>...

<state_snapshot>
    <overall_goal>
        <!-- A single, concise sentence describing the user's high-level objective. -->
    </overall_goal>

    <key_knowledge>
        <!-- Crucial facts, conventions, and constraints the agent must remember. -->
    </key_knowledge>

    <file_system_state>
        <!-- List files that have been created, read, modified, or deleted. -->
    </file_system_state>

    <recent_actions>
        <!-- A summary of the last few significant agent actions. -->
    </recent_actions>

    <current_plan>
        <!-- The agent's step-by-step plan. Mark completed steps. -->
    </current_plan>
</state_snapshot>
`.trim();
}
```

---

## 6. 會話管理

### 6.1 ChatRecordingService 完整分析

**檔案**: `/packages/core/src/services/chatRecordingService.ts`
**類別**: `ChatRecordingService` (Lines 111-495)

#### 6.1.1 資料結構

```typescript
// Lines 25-90
export interface TokensSummary {
  input: number;      // promptTokenCount
  output: number;     // candidatesTokenCount
  cached: number;     // cachedContentTokenCount
  thoughts?: number;  // thoughtsTokenCount
  tool?: number;      // toolUsePromptTokenCount
  total: number;      // totalTokenCount
}

export interface ToolCallRecord {
  id: string;
  name: string;
  args: Record<string, unknown>;
  result?: PartListUnion | null;
  status: Status;
  timestamp: string;
  displayName?: string;
  description?: string;
  resultDisplay?: string;
  renderOutputAsMarkdown?: boolean;
}

export interface ConversationRecord {
  sessionId: string;
  projectHash: string;
  startTime: string;
  lastUpdated: string;
  messages: MessageRecord[];
  summary?: string;
}
```

#### 6.1.2 初始化流程

```typescript
// Lines 130-178
initialize(resumedSessionData?: ResumedSessionData): void {
  try {
    if (resumedSessionData) {
      // 恢復現有會話
      this.conversationFile = resumedSessionData.filePath;
      this.sessionId = resumedSessionData.conversation.sessionId;

      this.updateConversation((conversation) => {
        conversation.sessionId = this.sessionId;
      });
      this.cachedLastConvData = null;
    } else {
      // 創建新會話
      const chatsDir = path.join(
        this.config.storage.getProjectTempDir(),
        'chats'
      );
      fs.mkdirSync(chatsDir, { recursive: true });

      const timestamp = new Date()
        .toISOString()
        .slice(0, 16)
        .replace(/:/g, '-');
      const filename = `${SESSION_FILE_PREFIX}${timestamp}-${this.sessionId.slice(0, 8)}.json`;
      this.conversationFile = path.join(chatsDir, filename);

      this.writeConversation({
        sessionId: this.sessionId,
        projectHash: this.projectHash,
        startTime: new Date().toISOString(),
        lastUpdated: new Date().toISOString(),
        messages: [],
      });
    }

    this.queuedThoughts = [];
    this.queuedTokens = null;
  } catch (error) {
    debugLogger.error('Error initializing chat recording service:', error);
    throw error;
  }
}
```

#### 6.1.3 訊息記錄

```typescript
// Lines 201-230
recordMessage(message: {
  model: string | undefined;
  type: ConversationRecordExtra['type'];
  content: PartListUnion;
}): void {
  if (!this.conversationFile) return;

  try {
    this.updateConversation((conversation) => {
      const msg = this.newMessage(message.type, message.content);

      if (msg.type === 'gemini') {
        // Gemini 訊息：附加排隊的思考和 tokens
        conversation.messages.push({
          ...msg,
          thoughts: this.queuedThoughts,
          tokens: this.queuedTokens,
          model: message.model,
        });
        this.queuedThoughts = [];
        this.queuedTokens = null;
      } else {
        conversation.messages.push(msg);
      }
    });
  } catch (error) {
    debugLogger.error('Error saving message to chat history.', error);
    throw error;
  }
}
```

#### 6.1.4 工具呼叫記錄

```typescript
// Lines 290-381
recordToolCalls(model: string, toolCalls: ToolCallRecord[]): void {
  if (!this.conversationFile) return;

  // 從 ToolRegistry 豐富工具呼叫資訊
  const toolRegistry = this.config.getToolRegistry();
  const enrichedToolCalls = toolCalls.map((toolCall) => {
    const toolInstance = toolRegistry.getTool(toolCall.name);
    return {
      ...toolCall,
      displayName: toolInstance?.displayName || toolCall.name,
      description: toolInstance?.description || '',
      renderOutputAsMarkdown: toolInstance?.isOutputMarkdown || false,
    };
  });

  try {
    this.updateConversation((conversation) => {
      const lastMsg = this.getLastMessage(conversation);

      // 如果最後訊息不是 Gemini 或有排隊的思考，創建新訊息
      if (!lastMsg || lastMsg.type !== 'gemini' || this.queuedThoughts.length > 0) {
        const newMsg: MessageRecord = {
          ...this.newMessage('gemini' as const, ''),
          type: 'gemini' as const,
          toolCalls: enrichedToolCalls,
          thoughts: this.queuedThoughts,
          model,
        };
        if (this.queuedThoughts.length > 0) {
          newMsg.thoughts = this.queuedThoughts;
          this.queuedThoughts = [];
        }
        if (this.queuedTokens) {
          newMsg.tokens = this.queuedTokens;
          this.queuedTokens = null;
        }
        conversation.messages.push(newMsg);
      } else {
        // 更新現有 Gemini 訊息
        if (!lastMsg.toolCalls) {
          lastMsg.toolCalls = [];
        }

        // 更新現有工具呼叫
        lastMsg.toolCalls = lastMsg.toolCalls.map((toolCall) => {
          const incomingToolCall = toolCalls.find((tc) => tc.id === toolCall.id);
          return incomingToolCall ? { ...toolCall, ...incomingToolCall } : toolCall;
        });

        // 添加新工具呼叫
        for (const toolCall of enrichedToolCalls) {
          if (!lastMsg.toolCalls.find((tc) => tc.id === toolCall.id)) {
            lastMsg.toolCalls.push(toolCall);
          }
        }
      }
    });
  } catch (error) {
    debugLogger.error('Error adding tool call to message.', error);
    throw error;
  }
}
```

### 6.2 儲存路徑結構

```
~/.gemini/
└── tmp/
    └── <PROJECT_HASH>/
        └── chats/
            └── session-2025-01-15T10-30-<SESSION_ID_PREFIX>.json
```

### 6.3 會話檔案格式

```json
{
  "sessionId": "uuid-v4",
  "projectHash": "sha256-hash",
  "startTime": "2025-01-15T10:30:00.000Z",
  "lastUpdated": "2025-01-15T10:35:00.000Z",
  "messages": [
    {
      "id": "uuid-v4",
      "timestamp": "2025-01-15T10:30:00.000Z",
      "type": "user",
      "content": "使用者訊息內容"
    },
    {
      "id": "uuid-v4",
      "timestamp": "2025-01-15T10:30:01.000Z",
      "type": "gemini",
      "content": "模型回應內容",
      "thoughts": [
        {
          "subject": "思考主題",
          "description": "思考描述",
          "timestamp": "2025-01-15T10:30:01.000Z"
        }
      ],
      "tokens": {
        "input": 1000,
        "output": 500,
        "cached": 200,
        "thoughts": 100,
        "tool": 50,
        "total": 1850
      },
      "toolCalls": [
        {
          "id": "call-id",
          "name": "read_file",
          "args": { "path": "/path/to/file" },
          "result": "檔案內容",
          "status": "success",
          "timestamp": "2025-01-15T10:30:02.000Z",
          "displayName": "Read File",
          "description": "讀取檔案內容"
        }
      ],
      "model": "gemini-2.0-flash"
    }
  ],
  "summary": "會話摘要（可選）"
}
```

---

## 7. 串流處理

### 7.1 StreamEvent 類型系統

```typescript
// geminiChat.ts Lines 58-68
export enum StreamEventType {
  /** 來自 API 的常規內容 chunk */
  CHUNK = 'chunk',
  /** 重試信號，UI 應丟棄之前嘗試的部分內容 */
  RETRY = 'retry',
}

export type StreamEvent =
  | { type: StreamEventType.CHUNK; value: GenerateContentResponse }
  | { type: StreamEventType.RETRY };
```

### 7.2 非同步生成器模式

#### 7.2.1 生成器創建

```typescript
// sendMessageStream 中的生成器創建 (Lines 292-376)
const streamWithRetries = async function* (
  this: GeminiChat,
): AsyncGenerator<StreamEvent, void, void> {
  try {
    let lastError: unknown = new Error('Request failed after all retries.');
    const maxAttempts = 2;

    for (let attempt = 0; attempt < maxAttempts; attempt++) {
      let isConnectionPhase = true;
      try {
        // 重試時產出 RETRY 事件
        if (attempt > 0) {
          yield { type: StreamEventType.RETRY };
        }

        isConnectionPhase = true;
        const stream = await this.makeApiCallAndProcessStream(...);
        isConnectionPhase = false;

        // 產出每個 chunk
        for await (const chunk of stream) {
          yield { type: StreamEventType.CHUNK, value: chunk };
        }

        lastError = null;
        break;
      } catch (error) {
        // 錯誤處理...
      }
    }

    if (lastError) throw lastError;
  } finally {
    streamDoneResolver!();
  }
};

return streamWithRetries.call(this);
```

#### 7.2.2 生成器消費

```typescript
// Turn.run() 中的生成器消費 (Lines 222-351)
async *run(
  modelConfigKey: ModelConfigKey,
  req: PartListUnion,
  signal: AbortSignal,
): AsyncGenerator<ServerGeminiStreamEvent> {
  const responseStream = await this.chat.sendMessageStream(...);

  for await (const streamEvent of responseStream) {
    if (signal?.aborted) {
      yield { type: GeminiEventType.UserCancelled };
      return;
    }

    // 處理 RETRY 事件
    if (streamEvent.type === 'retry') {
      yield { type: GeminiEventType.Retry };
      continue;
    }

    // 處理 CHUNK 事件
    const resp = streamEvent.value;
    // ... 轉換為高階事件
  }
}
```

### 7.3 串流消費流程圖

```
┌──────────────────────────────────────────────────────────────────────┐
│                        串流消費流程                                    │
└──────────────────────────────────────────────────────────────────────┘

         ┌─────────────────┐
         │   API Response   │
         │  AsyncGenerator  │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ processStream   │
         │   Response()    │
         └────────┬────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
     ▼            ▼            ▼
┌─────────┐ ┌─────────┐ ┌─────────────┐
│ 思考    │ │ 內容    │ │ 工具呼叫    │
│ parts   │ │ parts   │ │ parts       │
└────┬────┘ └────┬────┘ └──────┬──────┘
     │            │             │
     ▼            ▼             ▼
┌─────────┐ ┌─────────┐ ┌─────────────┐
│ 記錄    │ │ 合併    │ │ 收集        │
│ 思考    │ │ 文字    │ │ 函數呼叫    │
└─────────┘ └─────────┘ └─────────────┘
                  │
                  ▼
         ┌─────────────────┐
         │   yield chunk   │
         │  (即時產出)      │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │   驗證完成      │
         │ (finishReason)  │
         └────────┬────────┘
                  │
         ┌───────┴───────┐
         │               │
         ▼               ▼
   ┌──────────┐   ┌──────────────┐
   │ 有效完成 │   │ 無效串流     │
   │ 加入歷史 │   │ 拋出錯誤     │
   └──────────┘   └──────────────┘
```

---

## 8. 錯誤處理模式

### 8.1 InvalidStreamError 定義與類型

```typescript
// geminiChat.ts Lines 184-198
export class InvalidStreamError extends Error {
  readonly type:
    | 'NO_FINISH_REASON'      // 無完成原因
    | 'NO_RESPONSE_TEXT'      // 無回應文字
    | 'MALFORMED_FUNCTION_CALL'; // 畸形函數呼叫

  constructor(
    message: string,
    type: 'NO_FINISH_REASON' | 'NO_RESPONSE_TEXT' | 'MALFORMED_FUNCTION_CALL',
  ) {
    super(message);
    this.name = 'InvalidStreamError';
    this.type = type;
  }
}
```

### 8.2 錯誤分類表

| 錯誤類型 | 狀態碼 | 可重試 | 處理策略 |
|---------|--------|--------|----------|
| `ECONNRESET` | - | 是 | 指數退避重試 |
| `ETIMEDOUT` | - | 是 | 指數退避重試 |
| `EPIPE` | - | 是 | 指數退避重試 |
| `ENOTFOUND` | - | 是 | 指數退避重試 |
| `fetch failed` | - | 是 | 指數退避重試 |
| `ApiError` | 400 | 否 | 立即失敗 |
| `ApiError` | 429 | 是 | 退避或回退 |
| `ApiError` | 5xx | 是 | 指數退避重試 |
| `TerminalQuotaError` | 429 | 否 | 回退到備用模型 |
| `RetryableQuotaError` | 429 | 是 | 使用 API 提供的延遲 |
| `ModelNotFoundError` | 404 | 否 | 回退到備用模型 |
| `InvalidStreamError` | - | 是* | 僅 Gemini 2 重試 |

### 8.3 工具錯誤類型

```typescript
// tools/tool-error.ts
export enum ToolErrorType {
  TOOL_NOT_REGISTERED = 'tool_not_registered',
  INVALID_TOOL_PARAMS = 'invalid_tool_params',
  EXECUTION_FAILED = 'execution_failed',
  UNHANDLED_EXCEPTION = 'unhandled_exception',
  STOP_EXECUTION = 'stop_execution',  // Hook 請求停止
}
```

### 8.4 錯誤處理流程

```typescript
// 工具執行錯誤處理 (coreToolScheduler.ts Lines 1005-1028)
try {
  const toolResult: ToolResult = await promise;

  if (signal.aborted) {
    this.setStatusInternal(callId, 'cancelled', signal,
      'User cancelled tool execution.');
  } else if (toolResult.error === undefined) {
    // 成功
    this.setStatusInternal(callId, 'success', signal, successResponse);
  } else {
    // 工具返回錯誤
    this.setStatusInternal(callId, 'error', signal,
      createErrorResponse(request, new Error(toolResult.error.message),
        toolResult.error.type));
  }
} catch (executionError: unknown) {
  if (signal.aborted) {
    this.setStatusInternal(callId, 'cancelled', signal,
      'User cancelled tool execution.');
  } else {
    // 未捕獲異常
    this.setStatusInternal(callId, 'error', signal,
      createErrorResponse(request,
        executionError instanceof Error ? executionError : new Error(String(executionError)),
        ToolErrorType.UNHANDLED_EXCEPTION));
  }
}
```

---

## 9. 關鍵架構模式

### 9.1 非同步生成器模式

非同步生成器是整個串流系統的基礎，它允許：
- **惰性求值**: 只在需要時產生值
- **背壓控制**: 消費者控制處理速度
- **清潔取消**: 通過 `return()` 或 `throw()` 中斷

```typescript
// 模式示例
async function* createStream(): AsyncGenerator<StreamEvent> {
  try {
    for (let attempt = 0; attempt < maxAttempts; attempt++) {
      const response = await fetchData();
      for await (const chunk of response) {
        yield { type: 'chunk', value: chunk };
      }
    }
  } finally {
    // 清理資源
    cleanup();
  }
}

// 消費
const stream = createStream();
try {
  for await (const event of stream) {
    process(event);
    if (shouldStop) break;  // 觸發 finally
  }
} finally {
  // 確保清理
}
```

### 9.2 狀態機模式

CoreToolScheduler 使用顯式狀態機管理工具呼叫生命週期：

```typescript
// 狀態轉換函數
private setStatusInternal(
  targetCallId: string,
  newStatus: Status,
  signal: AbortSignal,
  auxiliaryData?: unknown,
): void {
  this.toolCalls = this.toolCalls.map((currentCall) => {
    // 防止從終端狀態轉換
    if (currentCall.status === 'success' ||
        currentCall.status === 'error' ||
        currentCall.status === 'cancelled') {
      return currentCall;
    }

    // 根據新狀態創建新物件
    switch (newStatus) {
      case 'success':
        return { ...currentCall, status: 'success', response: auxiliaryData };
      case 'error':
        return { ...currentCall, status: 'error', response: auxiliaryData };
      // ... 其他狀態
    }
  });

  this.notifyToolCallsUpdate();
}
```

### 9.3 歷史策展模式

雙軌歷史系統確保 API 兼容性：

```
綜合歷史 (Debug/Audit)          策展歷史 (API)
┌──────────────────────┐       ┌──────────────────────┐
│ [user] 訊息 1        │       │ [user] 訊息 1        │
│ [model] 回應 1       │   →   │ [model] 回應 1       │
│ [user] 訊息 2        │       │ [user] 訊息 2        │
│ [model] 無效回應 ✗   │       │                      │ ← 被過濾
│ [user] 訊息 3        │       │ [user] 訊息 3        │
│ [model] 回應 3       │       │ [model] 回應 3       │
└──────────────────────┘       └──────────────────────┘
```

### 9.4 Hook 擴展點模式

```
┌─────────────────────────────────────────────────────────────────┐
│                      Hook 生命週期                               │
└─────────────────────────────────────────────────────────────────┘

API 呼叫前:
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ BeforeModel │ --> │ BeforeToolSelect │ --> │ API 呼叫        │
└─────────────┘     └──────────────────┘     └─────────────────┘
      │                     │
      │ 可修改:             │ 可修改:
      │ - config            │ - toolConfig
      │ - contents          │ - tools
      │ 可阻擋             │
      ▼                     ▼

API 呼叫後:
┌─────────────────┐
│   AfterModel    │
└─────────────────┘
      │
      │ 可修改:
      │ - response
      ▼

工具執行前後:
┌─────────────┐     ┌───────────────┐     ┌─────────────┐
│ BeforeTool  │ --> │ 工具執行       │ --> │ AfterTool   │
└─────────────┘     └───────────────┘     └─────────────┘
      │                                          │
      │ 可修改: tool_input                      │ 可添加: additional_context
      │ 可阻擋                                  │ 可阻擋
      │ 可停止執行 (STOP_EXECUTION)            │ 可停止執行
      ▼                                          ▼
```

---

## 10. 序列圖

### 10.1 完整訊息發送流程

```
使用者                    Turn              GeminiChat           API
  │                        │                    │                  │
  │ sendMessage(message)   │                    │                  │
  │───────────────────────>│                    │                  │
  │                        │                    │                  │
  │                        │ sendMessageStream()│                  │
  │                        │───────────────────>│                  │
  │                        │                    │                  │
  │                        │                    │ await sendPromise│
  │                        │                    │ (序列化等待)      │
  │                        │                    │                  │
  │                        │                    │ recordUserMessage│
  │                        │                    │                  │
  │                        │                    │ history.push()   │
  │                        │                    │                  │
  │                        │                    │ getHistory(true) │
  │                        │                    │ (策展)           │
  │                        │                    │                  │
  │                        │                    │   ┌─────────────┐│
  │                        │                    │   │ 重試迴圈    ││
  │                        │                    │   │ attempt=0   ││
  │                        │                    │   └──────┬──────┘│
  │                        │                    │          │       │
  │                        │                    │ ┌────────▼───────┤
  │                        │                    │ │ FireHooks      │
  │                        │                    │ │ BeforeModel    │
  │                        │                    │ │ BeforeToolSel  │
  │                        │                    │ └────────┬───────┤
  │                        │                    │          │       │
  │                        │                    │          │generateContentStream
  │                        │                    │          │──────>│
  │                        │                    │          │       │
  │                        │                    │          │<chunks│
  │                        │                    │<─────────│       │
  │                        │                    │          │       │
  │                        │ yield CHUNK        │ processStreamResponse
  │                        │<───────────────────│          │       │
  │                        │                    │          │       │
  │                        │ 轉換為高階事件      │          │       │
  │<─ Content/Thought/Tool │                    │          │       │
  │                        │                    │          │       │
  │                        │                    │ ┌────────▼───────┤
  │                        │                    │ │ 驗證串流       │
  │                        │                    │ │ hasToolCall?   │
  │                        │                    │ │ finishReason?  │
  │                        │                    │ │ responseText?  │
  │                        │                    │ └────────┬───────┤
  │                        │                    │          │       │
  │                        │                    │ [無效] InvalidStreamError
  │                        │                    │   ┌──────▼──────┐│
  │                        │                    │   │ attempt++   ││
  │                        │ yield RETRY        │   │ delay       ││
  │                        │<───────────────────│   │ continue    ││
  │                        │                    │   └─────────────┘│
  │<─ Retry                │                    │                  │
  │                        │                    │                  │
  │                        │                    │ [有效] history.push(model)
  │                        │                    │                  │
  │                        │ yield Finished     │ streamDoneResolver()
  │                        │<───────────────────│                  │
  │<─ Finished             │                    │                  │
  │                        │                    │                  │
```

### 10.2 工具執行協調流程

```
模型回應                CoreToolScheduler                工具                 使用者
   │                          │                          │                      │
   │ ToolCallRequest          │                          │                      │
   │─────────────────────────>│                          │                      │
   │                          │                          │                      │
   │                          │ schedule()               │                      │
   │                          │ ┌──────────────────┐     │                      │
   │                          │ │ 檢查 isRunning   │     │                      │
   │                          │ │ isScheduling     │     │                      │
   │                          │ └────────┬─────────┘     │                      │
   │                          │          │               │                      │
   │                          │ [正在執行] 加入佇列      │                      │
   │                          │          │               │                      │
   │                          │ [空閒] _schedule()       │                      │
   │                          │ ┌────────▼─────────┐     │                      │
   │                          │ │ 查找工具實例     │     │                      │
   │                          │ │ 建構 invocation  │     │                      │
   │                          │ │ 狀態 = validating│     │                      │
   │                          │ └────────┬─────────┘     │                      │
   │                          │          │               │                      │
   │                          │ _processNextInQueue()    │                      │
   │                          │ ┌────────▼─────────┐     │                      │
   │                          │ │shouldConfirmExec │     │                      │
   │                          │ └────────┬─────────┘     │                      │
   │                          │          │               │                      │
   │                          │ [需要確認]                │                      │
   │                          │ ┌────────▼─────────┐     │                      │
   │                          │ │isAutoApproved?   │     │                      │
   │                          │ └────────┬─────────┘     │                      │
   │                          │          │               │                      │
   │                          │ [未自動批准]              │ 確認請求              │
   │                          │──────────┼───────────────┼─────────────────────>│
   │                          │          │               │                      │
   │                          │ 狀態 = awaiting_approval │   使用者決定         │
   │                          │<─────────┼───────────────┼──────────────────────│
   │                          │          │               │                      │
   │                          │ handleConfirmationResponse                      │
   │                          │ ┌────────▼─────────┐     │                      │
   │                          │ │ outcome處理      │     │                      │
   │                          │ │ ProceedOnce     │     │                      │
   │                          │ │ ProceedAlways   │     │                      │
   │                          │ │ Cancel          │     │                      │
   │                          │ │ ModifyWithEditor│     │                      │
   │                          │ └────────┬─────────┘     │                      │
   │                          │          │               │                      │
   │                          │ [批准] 狀態 = scheduled  │                      │
   │                          │          │               │                      │
   │                          │ attemptExecutionOfScheduledCalls               │
   │                          │ ┌────────▼─────────┐     │                      │
   │                          │ │狀態 = executing  │     │                      │
   │                          │ │設置 liveOutput   │     │                      │
   │                          │ │callback          │     │                      │
   │                          │ └────────┬─────────┘     │                      │
   │                          │          │               │                      │
   │                          │          │ execute()     │                      │
   │                          │          │──────────────>│                      │
   │                          │          │               │                      │
   │                          │          │<──liveOutput──│                      │
   │                          │<─────────│               │                      │
   │ OutputUpdate             │          │               │                      │
   │<─────────────────────────│          │               │                      │
   │                          │          │               │                      │
   │                          │          │<──result──────│                      │
   │                          │          │               │                      │
   │                          │ ┌────────▼─────────┐     │                      │
   │                          │ │ 處理結果         │     │                      │
   │                          │ │ 成功: success    │     │                      │
   │                          │ │ 失敗: error      │     │                      │
   │                          │ │ 取消: cancelled  │     │                      │
   │                          │ └────────┬─────────┘     │                      │
   │                          │          │               │                      │
   │                          │ checkAndNotifyCompletion │                      │
   │                          │ ┌────────▼─────────┐     │                      │
   │                          │ │ 更新完成列表     │     │                      │
   │                          │ │ 清空活動列表     │     │                      │
   │                          │ │ 處理下一個佇列   │     │                      │
   │                          │ └────────┬─────────┘     │                      │
   │                          │          │               │                      │
   │                          │ onAllToolCallsComplete   │                      │
   │<─────────────────────────│          │               │                      │
   │                          │          │               │                      │
   │ [佇列有更多請求]          │ 處理下一批次              │                      │
   │                          │──────────>               │                      │
```

---

## 11. 關鍵行號參考

### 11.1 baseLlmClient.ts

| 功能 | 行號範圍 | 說明 |
|------|----------|------|
| 類別定義 | 101-106 | BaseLlmClient 建構子 |
| generateJson | 108-159 | JSON 生成方法 |
| generateEmbedding | 161-194 | 嵌入向量生成 |
| cleanJsonResponse | 196-207 | JSON 回應清理 |
| generateContent | 209-238 | 一般內容生成 |
| _generateWithRetry | 240-338 | 核心重試邏輯 |

### 11.2 geminiChat.ts

| 功能 | 行號範圍 | 說明 |
|------|----------|------|
| StreamEventType | 58-68 | 串流事件類型定義 |
| InvalidStreamError | 184-198 | 無效串流錯誤類別 |
| GeminiChat 類別 | 207-227 | 類別定義與建構子 |
| sendMessageStream | 258-379 | 訊息串流發送 |
| makeApiCallAndProcessStream | 381-549 | API 呼叫與處理 |
| getHistory | 574-581 | 歷史獲取 |
| processStreamResponse | 699-817 | 串流回應處理與驗證 |
| recordCompletedToolCalls | 834-855 | 工具呼叫記錄 |

### 11.3 coreToolScheduler.ts

| 功能 | 行號範圍 | 說明 |
|------|----------|------|
| CoreToolScheduler 類別 | 114-176 | 類別定義與狀態 |
| setStatusInternal | 178-350 | 狀態轉換核心方法 |
| buildInvocation | 396-408 | 調用物件建構 |
| schedule | 410-450 | 公開調度入口 |
| cancelAll | 452-481 | 取消所有呼叫 |
| _schedule | 483-554 | 內部調度邏輯 |
| _processNextInQueue | 556-707 | 佇列處理 |
| handleConfirmationResponse | 709-781 | 確認回應處理 |
| attemptExecutionOfScheduledCalls | 831-1034 | 工具執行 |
| checkAndNotifyCompletion | 1036-1099 | 完成檢查與通知 |
| isAutoApproved | 1164-1178 | 自動批准判定 |

### 11.4 turn.ts

| 功能 | 行號範圍 | 說明 |
|------|----------|------|
| GeminiEventType | 52-69 | 事件類型枚舉 |
| 事件類型定義 | 71-207 | 各種事件介面 |
| Turn 類別 | 210-220 | 類別定義 |
| run 方法 | 222-351 | 主執行流程 |
| handlePendingFunctionCall | 353-376 | 函數呼叫處理 |

### 11.5 chatRecordingService.ts

| 功能 | 行號範圍 | 說明 |
|------|----------|------|
| TokensSummary | 25-32 | Token 摘要介面 |
| ToolCallRecord | 46-58 | 工具呼叫記錄介面 |
| ConversationRecord | 83-90 | 對話記錄介面 |
| ChatRecordingService | 111-495 | 服務類別 |
| initialize | 130-178 | 初始化 |
| recordMessage | 201-230 | 訊息記錄 |
| recordThought | 235-247 | 思考記錄 |
| recordMessageTokens | 252-284 | Token 記錄 |
| recordToolCalls | 290-381 | 工具呼叫記錄 |

### 11.6 prompts.ts

| 功能 | 行號範圍 | 說明 |
|------|----------|------|
| resolvePathFromEnv | 30-78 | 環境變數路徑解析 |
| getCoreSystemPrompt | 80-386 | 核心系統提示生成 |
| getCompressionPrompt | 393-451 | 壓縮提示生成 |

### 11.7 retry.ts

| 功能 | 行號範圍 | 說明 |
|------|----------|------|
| RetryOptions | 21-35 | 重試選項介面 |
| DEFAULT_RETRY_OPTIONS | 37-42 | 預設重試配置 |
| isRetryableError | 85-116 | 可重試錯誤判定 |
| retryWithBackoff | 125-290 | 重試核心邏輯 |

---

## 總結

Core 套件是 Gemini CLI 的核心架構，其設計展現了以下關鍵特點：

1. **分層架構**: 清晰的職責分離，從 API 客戶端到會話管理
2. **彈性重試**: 多層重試機制，包括網路錯誤和內容錯誤
3. **狀態機管理**: 工具調度器使用顯式狀態機確保正確的生命週期管理
4. **非同步串流**: 充分利用非同步生成器實現即時回應處理
5. **可擴展 Hook 系統**: 在關鍵點提供擴展能力
6. **完整持久化**: 會話記錄確保可恢復性和可追蹤性

這些設計使得 Gemini CLI 能夠提供穩定、可靠且功能豐富的 AI 對話體驗。
