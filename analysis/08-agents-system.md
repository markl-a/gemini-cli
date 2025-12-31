# 代理系統深度分析報告

## 1. 代理類型與註冊表

### 1.1 代理類型層次結構

**檔案路徑:** `/packages/core/src/agents/types.ts`

```typescript
// Lines 66-98: 代理類型定義
export interface LocalAgentDefinition<TOutput extends z.ZodTypeAny = z.ZodUnknown>
  extends BaseAgentDefinition<TOutput> {
  kind: 'local';
  promptConfig: PromptConfig;
  modelConfig: ModelConfig;
  runConfig: RunConfig;
  toolConfig?: ToolConfig;
  processOutput?: (output: z.infer<TOutput>) => string;
}

export interface RemoteAgentDefinition<TOutput extends z.ZodTypeAny = z.ZodUnknown>
  extends BaseAgentDefinition<TOutput> {
  kind: 'remote';
  agentCardUrl: string;
}
```

### 1.2 關鍵代理組件

| 組件 | 行數 | 用途 |
|------|------|------|
| InputConfig | 132-151 | 定義驗證的輸入參數 |
| OutputConfig | 156-173 | 定義預期輸出結構 |
| PromptConfig | 103-120 | 系統提示、初始訊息 |
| ModelConfig | 178-183 | 模型選擇、溫度 |
| RunConfig | 188-193 | 執行約束 |
| AgentTerminateMode | 18-25 | 終止模式列舉 |

### 1.3 終止模式

```typescript
export enum AgentTerminateMode {
  ERROR = 'ERROR',
  TIMEOUT = 'TIMEOUT',
  GOAL = 'GOAL',
  MAX_TURNS = 'MAX_TURNS',
  ABORTED = 'ABORTED',
  ERROR_NO_COMPLETE_TASK_CALL = 'ERROR_NO_COMPLETE_TASK_CALL',
}
```

---

## 2. 代理註冊表架構

**檔案路徑:** `/packages/core/src/agents/registry.ts`

### 2.1 AgentRegistry 類別 (Lines 38-104)

```typescript
export class AgentRegistry {
  private readonly agents = new Map<string, AgentDefinition<any>>();

  async initialize(): Promise<void> {
    this.loadBuiltInAgents();

    // 載入使用者級別代理: ~/.gemini/agents/
    const userAgentsDir = Storage.getUserAgentsDir();
    const userAgents = await loadAgentsFromDirectory(userAgentsDir);

    // 載入專案級別代理: .gemini/agents/ (如果信任)
    const projectAgentsDir = this.config.storage.getProjectAgentsDir();
    const projectAgents = await loadAgentsFromDirectory(projectAgentsDir);
  }
}
```

### 2.2 註冊表載入順序 (Lines 47-104)

1. 內建代理 (CodebaseInvestigatorAgent, IntrospectionAgent)
2. 使用者級別代理 (~/.gemini/agents/)
3. 專案級別代理 (.gemini/agents/ - 如果資料夾信任)

### 2.3 內建代理

**CodebaseInvestigatorAgent** (`codebase_investigator`)
- 檔案: `/packages/core/src/agents/codebase-investigator.ts` (Lines 44-154)
- 專門用於深度程式碼庫分析
- 使用唯讀工具: LS, READ_FILE, GLOB, GREP
- 輸出結構化報告

**IntrospectionAgent** (`introspection_agent`)
- 檔案: `/packages/core/src/agents/introspection-agent.ts` (Lines 25-85)
- 回答關於 Gemini CLI 本身的問題
- 使用 GetInternalDocsTool

---

## 3. 本地代理執行

### 3.1 LocalAgentExecutor 生命週期

**檔案路徑:** `/packages/core/src/agents/local-executor.ts`

**工廠方法 (Lines 75-138):**
```typescript
export class LocalAgentExecutor<TOutput extends z.ZodTypeAny> {
  static async create<TOutput extends z.ZodTypeAny>(
    definition: LocalAgentDefinition<TOutput>,
    runtimeContext: Config,
    onActivity?: ActivityCallback,
  ): Promise<LocalAgentExecutor<TOutput>> {
    // 為此代理創建隔離的工具註冊表
    const agentToolRegistry = new ToolRegistry(runtimeContext);
    const parentToolRegistry = runtimeContext.getToolRegistry();

    // 從定義註冊工具
    if (definition.toolConfig) {
      for (const toolRef of definition.toolConfig.tools) {
        if (typeof toolRef === 'string') {
          const toolFromParent = parentToolRegistry.getTool(toolRef);
          if (toolFromParent) {
            agentToolRegistry.registerTool(toolFromParent);
          }
        }
      }
    }
  }
}
```

### 3.2 主要執行迴圈

**Lines 353-547: `run` 方法**

```
run()
  ├─ 設置: 創建超時、初始化聊天物件
  ├─ 迴圈: while (true)
  │  ├─ 檢查終止 (max_turns)
  │  ├─ 檢查中止信號
  │  ├─ executeTurn():
  │  │  ├─ tryCompressChat()
  │  │  ├─ callModel(): 從模型獲取函數呼叫
  │  │  ├─ processFunctionCalls(): 執行工具和 complete_task
  │  │  └─ 返回: 'continue' 或 'stop'
  │  └─ 使用工具回應更新 currentMessage
  │
  ├─ 恢復區塊 (Lines 430-476):
  │  └─ 如果可終止原因 (TIMEOUT/MAX_TURNS/ERROR_NO_COMPLETE_TASK_CALL)
  │     └─ executeFinalWarningTurn(): 1 分鐘寬限期
  │
  └─ 返回 OutputObject
```

### 3.3 工具執行管理

**Lines 685-933: processFunctionCalls()**

```typescript
// 帶有並行執行的順序處理
const toolExecutionPromises: Array<Promise<Part[] | void>> = [];
const syncResponseParts: Part[] = [];

for (const functionCall of functionCalls) {
  if (functionCall.name === TASK_COMPLETE_TOOL_NAME) {
    // 同步完成處理與輸出驗證
    if (outputConfig) {
      const validationResult = outputConfig.schema.safeParse(outputValue);
      if (!validationResult.success) {
        taskCompleted = false; // 撤銷完成
      }
    }
  } else {
    // 非同步工具執行
    const executionPromise = (async () => {
      const agentContext = Object.create(this.runtimeContext);
      agentContext.getToolRegistry = () => this.toolRegistry;
      agentContext.getApprovalMode = () => ApprovalMode.YOLO;

      const { response: toolResponse } = await executeToolCall(...);
    })();
    toolExecutionPromises.push(executionPromise);
  }
}

// 等待所有工具執行完成
const asyncResults = await Promise.all(toolExecutionPromises);
```

---

## 4. 遠端代理支援 (A2A 協議)

### 4.1 A2A 客戶端管理

**檔案路徑:** `/packages/core/src/agents/a2a-client-manager.ts`

```typescript
// Lines 27-44: 單例模式實現
export class A2AClientManager {
  private static instance: A2AClientManager;
  private clients = new Map<string, Client>();
  private agentCards = new Map<string, AgentCard>();

  static getInstance(): A2AClientManager {
    if (!A2AClientManager.instance) {
      A2AClientManager.instance = new A2AClientManager();
    }
    return A2AClientManager.instance;
  }
}
```

### 4.2 代理載入與通訊

**Lines 62-100: loadAgent()**

```typescript
async loadAgent(name, agentCardUrl, authHandler?) {
  let fetchImpl = fetch;
  if (authHandler) {
    fetchImpl = createAuthenticatingFetchWithRetry(fetch, authHandler);
  }

  const resolver = new DefaultAgentCardResolver({ fetchImpl });
  const factory = new ClientFactory(options);
  const client = await factory.createFromUrl(agentCardUrl, '');
  const agentCard = await client.getAgentCard();

  this.clients.set(name, client);
  this.agentCards.set(name, agentCard);
}
```

### 4.3 訊息協議

**Lines 111-146: sendMessage()**

```typescript
async sendMessage(agentName, message, options?) {
  const client = this.clients.get(agentName);

  const messageParams: MessageSendParams = {
    message: {
      kind: 'message',
      role: 'user',
      messageId: uuidv4(),
      parts: [{ kind: 'text', text: message }],
      contextId: options?.contextId,
      taskId: options?.taskId,
    },
    configuration: {
      blocking: true,
    },
  };

  return await client.sendMessage(messageParams);
}
```

---

## 5. 子代理處理

### 5.1 SubAgent 工具包裝器

**檔案路徑:** `/packages/core/src/agents/subagent-tool-wrapper.ts`

```typescript
// Lines 24-56: SubagentToolWrapper
export class SubagentToolWrapper extends BaseDeclarativeTool<
  AgentInputs,
  ToolResult
> {
  constructor(definition, config, messageBus?) {
    const parameterSchema = convertInputConfigToJsonSchema(
      definition.inputConfig,
    );

    super(
      definition.name,
      definition.displayName ?? definition.name,
      definition.description,
      Kind.Think,
      parameterSchema,
      true, // isOutputMarkdown
      true, // canUpdateOutput
      messageBus,
    );
  }

  protected createInvocation(params: AgentInputs) {
    if (this.definition.kind === 'remote') {
      return new RemoteAgentInvocation(definition, params, this.messageBus);
    }

    return new LocalSubagentInvocation(
      definition,
      this.config,
      params,
      this.messageBus,
    );
  }
}
```

### 5.2 Delegate 工具

**檔案路徑:** `/packages/core/src/agents/delegate-to-agent-tool.ts`

```typescript
// Lines 26-138: DelegateToAgentTool - 區分聯合 Schema
export class DelegateToAgentTool extends BaseDeclarativeTool<
  DelegateParams,
  ToolResult
> {
  constructor(registry, config, messageBus?) {
    const definitions = registry.getAllDefinitions();

    // 使用 agent_name 鑑別器建構 schema
    const agentSchemas = definitions.map((def) => {
      const inputShape = {
        agent_name: z.literal(def.name).describe(def.description),
      };

      // 映射每個代理的輸入配置
      for (const [key, inputDef] of Object.entries(def.inputConfig.inputs)) {
        switch (inputDef.type) {
          case 'string': validator = z.string(); break;
          case 'number': validator = z.number(); break;
          case 'integer': validator = z.number().int(); break;
          case 'boolean': validator = z.boolean(); break;
          case 'string[]': validator = z.array(z.string()); break;
          case 'number[]': validator = z.array(z.number()); break;
        }

        inputShape[key] = validator.describe(inputDef.description);
      }

      return z.object(inputShape);
    });

    // 創建類型安全的區分聯合
    schema = z.discriminatedUnion('agent_name', agentSchemas);
  }
}
```

---

## 6. 代理配置與發現

### 6.1 TOML 配置格式

**檔案路徑:** `/packages/core/src/agents/toml-loader.ts`

```typescript
// Lines 26-42: 本地代理 TOML 結構
interface TomlLocalAgentDefinition {
  kind: 'local';
  description: string;
  tools?: string[];
  prompts: {
    system_prompt: string;
    query?: string;
  };
  model?: {
    model?: string;
    temperature?: number;
  };
  run?: {
    max_turns?: number;
    timeout_mins?: number;
  };
}

// Lines 44-48: 遠端代理 TOML 結構
interface TomlRemoteAgentDefinition {
  description?: string;
  kind: 'remote';
  agent_card_url: string;
}
```

### 6.2 配置驗證

**Lines 77-107: Zod 驗證 Schema**

```typescript
const localAgentSchema = z.object({
  kind: z.literal('local').optional().default('local'),
  name: nameSchema,
  description: z.string().min(1),
  display_name: z.string().optional(),
  tools: z.array(
    z.string().refine((val) => isValidToolName(val), {
      message: 'Invalid tool name',
    }),
  ).optional(),
  prompts: z.object({
    system_prompt: z.string().min(1),
    query: z.string().optional(),
  }),
  model: z.object({
    model: z.string().optional(),
    temperature: z.number().optional(),
  }).optional(),
  run: z.object({
    max_turns: z.number().int().positive().optional(),
    timeout_mins: z.number().int().positive().optional(),
  }).optional(),
}).strict();
```

### 6.3 代理發現程序

**Lines 297-354: loadAgentsFromDirectory()**

```typescript
export async function loadAgentsFromDirectory(dir) {
  const result = { agents: [], errors: [] };

  let dirEntries;
  try {
    dirEntries = await fs.readdir(dir, { withFileTypes: true });
  } catch (error) {
    if (error.code === 'ENOENT') return result;
  }

  // 過濾 .toml 檔案 (排除以 _ 開頭的)
  const files = dirEntries
    .filter(entry =>
      entry.isFile() &&
      entry.name.endsWith('.toml') &&
      !entry.name.startsWith('_'))
    .map(entry => entry.name);

  for (const file of files) {
    const filePath = path.join(dir, file);
    try {
      const tomls = await parseAgentToml(filePath);
      for (const toml of tomls) {
        const agent = tomlToAgentDefinition(toml);
        result.agents.push(agent);
      }
    } catch (error) {
      result.errors.push(new AgentLoadError(filePath, message));
    }
  }

  return result;
}
```

---

## 7. 活動事件與串流

**Lines 51-52, 1080-1093:**

```typescript
export type ActivityCallback = (activity: SubagentActivityEvent) => void;

interface SubagentActivityEvent {
  isSubagentActivityEvent: true;
  agentName: string;
  type: 'TOOL_CALL_START' | 'TOOL_CALL_END' | 'THOUGHT_CHUNK' | 'ERROR';
  data: Record<string, unknown>;
}

private emitActivity(type, data) {
  if (this.onActivity) {
    const event: SubagentActivityEvent = {
      isSubagentActivityEvent: true,
      agentName: this.definition.name,
      type,
      data,
    };
    this.onActivity(event);
  }
}
```

---

## 8. 超時與恢復機制

### 超時管理 (Lines 359-367)

```typescript
const { max_time_minutes } = this.definition.runConfig;
const timeoutController = new AbortController();
const timeoutId = setTimeout(
  () => timeoutController.abort(new Error('Agent timed out.')),
  max_time_minutes * 60 * 1000,
);

const combinedSignal = AbortSignal.any([signal, timeoutController.signal]);
```

### 恢復機制 (Lines 262-343)

```typescript
private async executeFinalWarningTurn(
  chat, turnCounter, reason, externalSignal
) {
  const gracePeriodMs = GRACE_PERIOD_MS; // 60 秒
  const graceTimeoutController = new AbortController();
  const graceTimeoutId = setTimeout(
    () => graceTimeoutController.abort(new Error('Grace period timed out.')),
    gracePeriodMs,
  );

  const turnResult = await this.executeTurn(
    chat, recoveryMessage, turnCounter, combinedSignal, graceTimeoutController.signal
  );

  if (turnResult.status === 'stop' &&
      turnResult.terminateReason === AgentTerminateMode.GOAL) {
    return turnResult.finalResult;
  }
}
```

---

## 9. 架構圖

### 代理生命週期流程

```
┌─────────────────────────────────────────────────────────────┐
│                   父代理執行                                  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  模型呼叫: delegate_to_agent(agent_name, ...inputs)          │
│                         ↓                                     │
│  DelegateToAgentTool.execute()                               │
│  ├─ 從 AgentRegistry 獲取代理定義                            │
│  ├─ 創建 SubagentToolWrapper                                 │
│  └─ SubagentToolWrapper.build() → createInvocation()        │
│                         ↓                                     │
│  ┌──────────────────────┴──────────────────────┐            │
│  ↓                                              ↓             │
│ LocalSubagentInvocation              RemoteAgentInvocation   │
│  │                                              │             │
│  ├─ LocalAgentExecutor.create()                │             │
│  │  ├─ 設置隔離的 ToolRegistry                 │             │
│  │  ├─ 從父註冊表複製工具                       │             │
│  │  └─ 配置代理實例                            │             │
│  │                                              │             │
│  └─ executor.run(inputs, signal)               │             │
│     ├─ 初始化 GeminiChat                       │             │
│     ├─ 主迴圈:                                 │             │
│     │  └─ executeTurn()                        │             │
│     │     ├─ callModel()                       │             │
│     │     └─ processFunctionCalls()            │             │
│     │                                          │             │
│     ├─ 終止檢查:                               │             │
│     │  └─ 如果可恢復 → executeFinalWarningTurn()│            │
│     │                                          │             │
│     └─ 返回 OutputObject                      │             │
│                                                │             │
└─────────────────────────────────────────────────────────────┘
```

### 代理註冊表發現流程

```
┌──────────────────────────────────────────┐
│     Config.initialize()                   │
├──────────────────────────────────────────┤
│                                           │
├─ AgentRegistry.initialize()               │
│  │                                        │
│  ├─ loadBuiltInAgents():                  │
│  │  ├─ CodebaseInvestigatorAgent          │
│  │  └─ IntrospectionAgent                 │
│  │                                        │
│  ├─ loadAgentsFromDirectory():            │
│  │  └─ ~/.gemini/agents/                  │
│  │     (forEach *.toml file)              │
│  │     ├─ parseAgentToml()                │
│  │     ├─ tomlToAgentDefinition()         │
│  │     └─ registerAgent()                 │
│  │                                        │
│  └─ loadAgentsFromDirectory():            │
│     └─ .gemini/agents/ (if trusted)       │
│                                           │
├─ DelegateToAgentTool 註冊:                │
│  ├─ 從所有代理 inputConfigs 建構          │
│  │  區分聯合 schema                        │
│  └─ 註冊工具到 ToolRegistry               │
│                                           │
└──────────────────────────────────────────┘
```

---

## 10. 關鍵安全與設計決策

1. **工具隔離**: 每個代理獲得自己隔離的 ToolRegistry
2. **無子代理遞迴**: 子代理不能使用 delegate_to_agent 工具
3. **YOLO 批准模式**: 代理以 ApprovalMode.YOLO 運行
4. **信任資料夾檢查**: 專案代理僅在資料夾信任時載入
5. **輸出驗證**: 使用 Zod schema 驗證 complete_task 輸出
6. **AbortSignal 傳播**: 取消信號流經整個執行鏈

---

## 檔案結構摘要

```
packages/core/src/agents/
├── types.ts                    # 類型定義
├── registry.ts                 # AgentRegistry: 發現、載入、註冊
├── local-executor.ts           # LocalAgentExecutor (1095 行)
├── local-invocation.ts         # LocalSubagentInvocation
├── remote-invocation.ts        # RemoteAgentInvocation (stub)
├── a2a-client-manager.ts       # A2A 協議客戶端
├── delegate-to-agent-tool.ts   # DelegateToAgentTool
├── subagent-tool-wrapper.ts    # SubagentToolWrapper
├── toml-loader.ts              # TOML 解析和驗證 (355 行)
├── schema-utils.ts             # JSON schema 轉換
├── utils.ts                    # 模板字串替換
├── codebase-investigator.ts    # 內建代理
└── introspection-agent.ts      # 內建代理
```
