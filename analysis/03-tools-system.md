# 工具系統深度分析報告

## 執行摘要

Gemini CLI 的工具系統是一個精心設計的模組化框架，用於在 AI 助手與本地環境之間建立安全且可控的互動橋樑。本報告深入分析工具系統的每個核心組件，包括架構設計、安全機制、執行流程和錯誤處理策略。

**關鍵指標:**
| 指標 | 數值 |
|------|------|
| 總實現檔案 | 49 個 TypeScript 檔案 |
| 核心工具數量 | 23+ 個 |
| 程式碼行數 | ~18,000+ 行 |
| 工具類別 | 9 種 (Kind 列舉) |
| 安全機制 | 3 層驗證架構 |
| 傳輸協議 | 3 種 MCP 傳輸層 |

---

## 1. 工具註冊與發現系統

### 1.1 ToolRegistry 核心類別

**檔案位置**: `/packages/core/src/tools/tool-registry.ts` (534 行)

`ToolRegistry` 是整個工具系統的核心管理器，負責工具的註冊、發現、排序和檢索。

#### 類別結構

```typescript
export class ToolRegistry {
  // 以工具名稱為鍵的映射表，包含所有已知工具（含非活躍工具）
  private allKnownTools: Map<string, AnyDeclarativeTool> = new Map();
  private config: Config;
  private messageBus?: MessageBus;

  constructor(config: Config) {
    this.config = config;
  }

  // 設置消息總線用於政策引擎整合
  setMessageBus(messageBus: MessageBus): void;
  getMessageBus(): MessageBus | undefined;

  // 工具註冊 - 支援重複註冊檢測
  registerTool(tool: AnyDeclarativeTool): void;

  // 工具排序 - 按優先順序組織
  sortTools(): void;

  // 工具發現 - 從命令行和 MCP 伺服器
  async discoverAllTools(): Promise<void>;

  // 工具檢索方法
  getFunctionDeclarations(): FunctionDeclaration[];
  getAllToolNames(): string[];
  getAllTools(): AnyDeclarativeTool[];
  getTool(name: string): AnyDeclarativeTool | undefined;
}
```

#### 工具排序優先順序

```
┌─────────────────────────────────────────────────────────┐
│                    工具排序機制                          │
├─────────────────────────────────────────────────────────┤
│  優先順序 0: 內建工具 (Built-in Tools)                   │
│  ├── ReadFileTool, WriteFileTool, EditTool             │
│  ├── GlobTool, GrepTool, ShellTool                     │
│  └── WebFetchTool, WebSearchTool, MemoryTool           │
├─────────────────────────────────────────────────────────┤
│  優先順序 1: 發現的工具 (DiscoveredTool)                 │
│  └── 通過 toolDiscoveryCommand 配置發現                 │
├─────────────────────────────────────────────────────────┤
│  優先順序 2: MCP 工具 (DiscoveredMCPTool)                │
│  └── 按伺服器名稱字母順序排序                            │
└─────────────────────────────────────────────────────────┘
```

#### 排序實現

```typescript
sortTools(): void {
  const getPriority = (tool: AnyDeclarativeTool): number => {
    if (tool instanceof DiscoveredMCPTool) return 2;
    if (tool instanceof DiscoveredTool) return 1;
    return 0; // Built-in
  };

  this.allKnownTools = new Map(
    Array.from(this.allKnownTools.entries()).sort((a, b) => {
      const toolA = a[1];
      const toolB = b[1];
      const priorityA = getPriority(toolA);
      const priorityB = getPriority(toolB);

      if (priorityA !== priorityB) {
        return priorityA - priorityB;
      }

      // MCP 工具按伺服器名稱排序
      if (priorityA === 2) {
        const serverA = (toolA as DiscoveredMCPTool).serverName;
        const serverB = (toolB as DiscoveredMCPTool).serverName;
        return serverA.localeCompare(serverB);
      }

      return 0; // 保持穩定排序
    }),
  );
}
```

### 1.2 DiscoveredTool 類別

**用途**: 包裝通過命令行發現的外部工具

```typescript
export class DiscoveredTool extends BaseDeclarativeTool<ToolParams, ToolResult> {
  private readonly originalName: string;

  constructor(
    private readonly config: Config,
    originalName: string,
    prefixedName: string,          // 格式: "discovered__<originalName>"
    description: string,
    parameterSchema: Record<string, unknown>,
    messageBus?: MessageBus,
  ) {
    // 自動生成完整描述，包含發現和調用命令
    const fullDescription = description + `
      This tool was discovered from the project by executing the command \`${discoveryCmd}\`.
      When called, this tool will execute the command \`${callCommand} ${originalName}\`.
    `;
    super(prefixedName, prefixedName, fullDescription, Kind.Other, ...);
  }
}
```

#### 工具發現流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    工具發現流程                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. discoverAllTools()                                           │
│    └── removeDiscoveredTools()  // 清除舊的發現工具              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. discoverAndRegisterToolsFromCommand()                        │
│    ├── 解析 toolDiscoveryCommand 配置                           │
│    ├── 使用 shell-quote 解析命令                                 │
│    └── 執行 spawn() 啟動子程序                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 串流處理                                                      │
│    ├── stdout 限制: 10MB                                        │
│    ├── stderr 限制: 10MB                                        │
│    └── 超過限制時終止程序                                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 解析 JSON 輸出                                                │
│    ├── 支援 function_declarations 陣列                          │
│    ├── 支援 functionDeclarations 陣列                           │
│    └── 支援直接的 FunctionDeclaration 物件                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 註冊工具                                                      │
│    ├── 為每個函數創建 DiscoveredTool 實例                        │
│    ├── 添加 "discovered__" 前綴                                  │
│    └── 調用 toolRegistry.registerTool()                         │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 工具活躍狀態管理

```typescript
private isActiveTool(
  tool: AnyDeclarativeTool,
  excludeTools?: Set<string>,
): boolean {
  excludeTools ??= this.config.getExcludeTools() ?? new Set([]);

  // 正規化類別名稱（移除前導底線）
  const normalizedClassName = tool.constructor.name.replace(/^_+/, '');

  // 可能的名稱列表
  const possibleNames = [tool.name, normalizedClassName];

  // MCP 工具支援完整限定名稱檢查
  if (tool instanceof DiscoveredMCPTool) {
    if (tool.name.startsWith(tool.getFullyQualifiedPrefix())) {
      possibleNames.push(
        tool.name.substring(tool.getFullyQualifiedPrefix().length),
      );
    } else {
      possibleNames.push(`${tool.getFullyQualifiedPrefix()}${tool.name}`);
    }
  }

  // 檢查是否被排除
  return !possibleNames.some((name) => excludeTools.has(name));
}
```

---

## 2. 基礎框架與工具介面

### 2.1 ToolInvocation 介面

**檔案位置**: `/packages/core/src/tools/tools.ts` (756 行)

`ToolInvocation` 定義了已驗證、準備執行的工具調用契約：

```typescript
export interface ToolInvocation<
  TParams extends object,
  TResult extends ToolResult,
> {
  // 已驗證的參數
  params: TParams;

  // 獲取工具操作的預執行描述
  getDescription(): string;

  // 確定工具將影響的檔案系統路徑
  toolLocations(): ToolLocation[];

  // 判斷是否需要確認執行
  shouldConfirmExecute(
    abortSignal: AbortSignal,
  ): Promise<ToolCallConfirmationDetails | false>;

  // 執行工具
  execute(
    signal: AbortSignal,
    updateOutput?: (output: string | AnsiOutput) => void,
    shellExecutionConfig?: ShellExecutionConfig,
  ): Promise<TResult>;
}
```

### 2.2 BaseToolInvocation 抽象類別

```typescript
export abstract class BaseToolInvocation<
  TParams extends object,
  TResult extends ToolResult,
> implements ToolInvocation<TParams, TResult> {

  constructor(
    readonly params: TParams,
    protected readonly messageBus?: MessageBus,
    readonly _toolName?: string,
    readonly _toolDisplayName?: string,
    readonly _serverName?: string,  // MCP 伺服器名稱
  ) {}

  // 政策引擎決策流程
  async shouldConfirmExecute(
    abortSignal: AbortSignal,
  ): Promise<ToolCallConfirmationDetails | false> {
    if (this.messageBus) {
      const decision = await this.getMessageBusDecision(abortSignal);

      if (decision === 'ALLOW') return false;
      if (decision === 'DENY') {
        throw new Error(`Tool execution denied by policy.`);
      }
      if (decision === 'ASK_USER') {
        return this.getConfirmationDetails(abortSignal);
      }
    }
    return this.getConfirmationDetails(abortSignal);
  }

  // 發布政策更新
  protected async publishPolicyUpdate(
    outcome: ToolConfirmationOutcome,
  ): Promise<void> {
    if (
      outcome === ToolConfirmationOutcome.ProceedAlways ||
      outcome === ToolConfirmationOutcome.ProceedAlwaysAndSave
    ) {
      if (this.messageBus && this._toolName) {
        const options = this.getPolicyUpdateOptions(outcome);
        await this.messageBus.publish({
          type: MessageBusType.UPDATE_POLICY,
          toolName: this._toolName,
          persist: outcome === ToolConfirmationOutcome.ProceedAlwaysAndSave,
          ...options,
        });
      }
    }
  }
}
```

### 2.3 DeclarativeTool 與 BaseDeclarativeTool

```typescript
// 聲明式工具基類
export abstract class DeclarativeTool<TParams, TResult>
  implements ToolBuilder<TParams, TResult> {

  constructor(
    readonly name: string,
    readonly displayName: string,
    readonly description: string,
    readonly kind: Kind,
    readonly parameterSchema: unknown,
    readonly isOutputMarkdown: boolean = true,
    readonly canUpdateOutput: boolean = false,
    readonly messageBus?: MessageBus,
    readonly extensionName?: string,    // 擴展名稱
    readonly extensionId?: string,      // 擴展 ID
  ) {}

  // 生成 FunctionDeclaration
  get schema(): FunctionDeclaration {
    return {
      name: this.name,
      description: this.description,
      parametersJsonSchema: this.parameterSchema,
    };
  }

  // 參數驗證
  validateToolParams(_params: TParams): string | null {
    return null;
  }

  // 構建調用實例
  abstract build(params: TParams): ToolInvocation<TParams, TResult>;
}

// 帶有自動驗證的聲明式工具
export abstract class BaseDeclarativeTool<TParams, TResult>
  extends DeclarativeTool<TParams, TResult> {

  build(params: TParams): ToolInvocation<TParams, TResult> {
    const validationError = this.validateToolParams(params);
    if (validationError) {
      throw new Error(validationError);
    }
    return this.createInvocation(params, this.messageBus, this.name, this.displayName);
  }

  override validateToolParams(params: TParams): string | null {
    // 第一層：JSON Schema 驗證
    const errors = SchemaValidator.validate(
      this.schema.parametersJsonSchema,
      params,
    );
    if (errors) return errors;

    // 第二層：業務邏輯驗證
    return this.validateToolParamValues(params);
  }

  protected validateToolParamValues(_params: TParams): string | null {
    return null;
  }

  protected abstract createInvocation(
    params: TParams,
    messageBus?: MessageBus,
    _toolName?: string,
    _toolDisplayName?: string,
  ): ToolInvocation<TParams, TResult>;
}
```

### 2.4 工具分類 Kind 列舉

```typescript
export enum Kind {
  Read = 'read',       // 檔案讀取操作
  Edit = 'edit',       // 檔案修改
  Delete = 'delete',   // 檔案刪除
  Move = 'move',       // 檔案移動
  Search = 'search',   // 內容/檔案搜尋
  Execute = 'execute', // Shell/子程序執行
  Think = 'think',     // 分析/推理
  Fetch = 'fetch',     // 遠端資源獲取
  Other = 'other',     // 其他
}

// 具有副作用的操作類型
export const MUTATOR_KINDS: Kind[] = [
  Kind.Edit,
  Kind.Delete,
  Kind.Move,
  Kind.Execute,
] as const;
```

### 2.5 ToolResult 介面

```typescript
export interface ToolResult {
  // LLM 歷史內容 - 工具執行的事實結果
  llmContent: PartListUnion;

  // 使用者顯示內容 - 友好的摘要或視覺化
  returnDisplay: ToolResultDisplay;

  // 錯誤標識 - 存在時表示工具調用失敗
  error?: {
    message: string;
    type?: ToolErrorType;
  };
}

// 顯示結果類型
export type ToolResultDisplay =
  | string
  | FileDiff
  | AnsiOutput
  | TodoList;
```

### 2.6 確認結果類型

```typescript
export enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed_once',              // 本次允許
  ProceedAlways = 'proceed_always',          // 本會話始終允許
  ProceedAlwaysAndSave = 'proceed_always_and_save',  // 永久允許並保存
  ProceedAlwaysServer = 'proceed_always_server',     // MCP 伺服器級別允許
  ProceedAlwaysTool = 'proceed_always_tool',         // MCP 工具級別允許
  ModifyWithEditor = 'modify_with_editor',   // 使用編輯器修改
  Cancel = 'cancel',                         // 取消
}

// 確認詳情類型
export type ToolCallConfirmationDetails =
  | ToolEditConfirmationDetails     // 編輯確認
  | ToolExecuteConfirmationDetails  // 執行確認
  | ToolMcpConfirmationDetails      // MCP 確認
  | ToolInfoConfirmationDetails;    // 資訊確認
```

---

## 3. 檔案工具詳細分析

### 3.1 ReadFile 工具

**檔案位置**: `/packages/core/src/tools/read-file.ts` (241 行)

#### 參數定義

```typescript
export interface ReadFileToolParams {
  file_path: string;   // 要讀取的檔案路徑
  offset?: number;     // 0-based 起始行號
  limit?: number;      // 最大讀取行數
}
```

#### 核心功能

```typescript
export class ReadFileTool extends BaseDeclarativeTool<ReadFileToolParams, ToolResult> {
  static readonly Name = 'read_file';

  constructor(private config: Config, messageBus?: MessageBus) {
    super(
      ReadFileTool.Name,
      'ReadFile',
      `Reads and returns the content of a specified file.
       Handles text, images (PNG, JPG, GIF, WEBP, SVG, BMP),
       audio files (MP3, WAV, AIFF, AAC, OGG, FLAC), and PDF files.
       For text files, it can read specific line ranges.`,
      Kind.Read,
      // ... schema
    );
  }
}
```

#### 驗證邏輯

```typescript
protected override validateToolParamValues(params: ReadFileToolParams): string | null {
  // 1. 非空檢查
  if (params.file_path.trim() === '') {
    return "The 'file_path' parameter must be non-empty.";
  }

  // 2. 工作區邊界檢查
  const workspaceContext = this.config.getWorkspaceContext();
  const projectTempDir = this.config.storage.getProjectTempDir();
  const resolvedPath = path.resolve(this.config.getTargetDir(), params.file_path);

  // 允許訪問臨時目錄
  const isWithinTempDir = resolvedPath.startsWith(resolvedProjectTempDir + path.sep);

  if (!workspaceContext.isPathWithinWorkspace(resolvedPath) && !isWithinTempDir) {
    return `File path must be within workspace directories or temp directory`;
  }

  // 3. 分頁參數驗證
  if (params.offset !== undefined && params.offset < 0) {
    return 'Offset must be a non-negative number';
  }
  if (params.limit !== undefined && params.limit <= 0) {
    return 'Limit must be a positive number';
  }

  // 4. 忽略模式檢查
  const fileFilteringOptions = this.config.getFileFilteringOptions();
  if (fileService.shouldIgnoreFile(resolvedPath, fileFilteringOptions)) {
    return `File path is ignored by configured ignore patterns.`;
  }

  return null;
}
```

#### 執行流程

```typescript
class ReadFileToolInvocation extends BaseToolInvocation<ReadFileToolParams, ToolResult> {
  async execute(): Promise<ToolResult> {
    // 調用檔案處理服務
    const result = await processSingleFileContent(
      this.resolvedPath,
      this.config.getTargetDir(),
      this.config.getFileSystemService(),
      this.params.offset,
      this.params.limit,
    );

    if (result.error) {
      return {
        llmContent: result.llmContent,
        returnDisplay: result.returnDisplay || 'Error reading file',
        error: { message: result.error, type: result.errorType },
      };
    }

    // 處理截斷情況
    let llmContent: PartUnion;
    if (result.isTruncated) {
      const [start, end] = result.linesShown!;
      const total = result.originalLineCount!;
      llmContent = `
IMPORTANT: The file content has been truncated.
Status: Showing lines ${start}-${end} of ${total} total lines.
Action: Use 'offset' and 'limit' parameters to read more.
--- FILE CONTENT (truncated) ---
${result.llmContent}`;
    } else {
      llmContent = result.llmContent || '';
    }

    // 記錄遙測
    logFileOperation(this.config, new FileOperationEvent(
      READ_FILE_TOOL_NAME,
      FileOperation.READ,
      lines,
      mimetype,
      extension,
      programming_language,
    ));

    return { llmContent, returnDisplay: result.returnDisplay || '' };
  }
}
```

### 3.2 WriteFile 工具

**檔案位置**: `/packages/core/src/tools/write-file.ts` (527 行)

#### 參數定義

```typescript
export interface WriteFileToolParams {
  file_path: string;              // 要寫入的檔案路徑
  content: string;                // 寫入內容
  modified_by_user?: boolean;     // 使用者修改標記
  ai_proposed_content?: string;   // 原始 AI 提案內容
}
```

#### 核心功能：內容校正

```typescript
export async function getCorrectedFileContent(
  config: Config,
  filePath: string,
  proposedContent: string,
  abortSignal: AbortSignal,
): Promise<GetCorrectedFileContentResult> {
  let originalContent = '';
  let fileExists = false;
  let correctedContent = proposedContent;

  try {
    originalContent = await config.getFileSystemService().readTextFile(filePath);
    fileExists = true;
  } catch (err) {
    if (isNodeError(err) && err.code === 'ENOENT') {
      fileExists = false;
    } else {
      return { originalContent, correctedContent, fileExists, error: { message, code } };
    }
  }

  if (fileExists) {
    // 現有檔案：使用 LLM 進行編輯校正
    const { params: correctedParams } = await ensureCorrectEdit(
      filePath,
      originalContent,
      {
        old_string: originalContent,
        new_string: proposedContent,
        file_path: filePath,
      },
      config.getGeminiClient(),
      config.getBaseLlmClient(),
      abortSignal,
    );
    correctedContent = correctedParams.new_string;
  } else {
    // 新檔案：使用 LLM 校正內容
    correctedContent = await ensureCorrectFileContent(
      proposedContent,
      config.getBaseLlmClient(),
      abortSignal,
    );
  }

  return { originalContent, correctedContent, fileExists };
}
```

#### ModifiableDeclarativeTool 實現

```typescript
export class WriteFileTool
  extends BaseDeclarativeTool<WriteFileToolParams, ToolResult>
  implements ModifiableDeclarativeTool<WriteFileToolParams> {

  getModifyContext(abortSignal: AbortSignal): ModifyContext<WriteFileToolParams> {
    return {
      getFilePath: (params) => params.file_path,

      getCurrentContent: async (params) => {
        const result = await getCorrectedFileContent(
          this.config, params.file_path, params.content, abortSignal
        );
        return result.originalContent;
      },

      getProposedContent: async (params) => {
        const result = await getCorrectedFileContent(
          this.config, params.file_path, params.content, abortSignal
        );
        return result.correctedContent;
      },

      createUpdatedParams: (oldContent, modifiedContent, originalParams) => ({
        ...originalParams,
        ai_proposed_content: originalParams.content,
        content: modifiedContent,
        modified_by_user: true,
      }),
    };
  }
}
```

### 3.3 Edit 工具

**檔案位置**: `/packages/core/src/tools/edit.ts` (630 行)

#### 參數定義

```typescript
export interface EditToolParams {
  file_path: string;              // 要修改的檔案
  old_string: string;             // 要替換的文字（空字串表示創建新檔案）
  new_string: string;             // 替換後的文字
  expected_replacements?: number; // 預期替換次數，預設為 1
  modified_by_user?: boolean;     // 使用者修改標記
  ai_proposed_content?: string;   // 原始 AI 提案
}
```

#### 編輯計算流程

```typescript
private async calculateEdit(
  params: EditToolParams,
  abortSignal: AbortSignal,
): Promise<CalculatedEdit> {
  const expectedReplacements = params.expected_replacements ?? 1;
  let currentContent: string | null = null;
  let fileExists = false;
  let isNewFile = false;

  // 1. 讀取檔案並正規化行結尾
  try {
    currentContent = await this.config.getFileSystemService()
      .readTextFile(this.resolvedPath);
    currentContent = currentContent.replace(/\r\n/g, '\n');
    fileExists = true;
  } catch (err) {
    if (!isNodeError(err) || err.code !== 'ENOENT') throw err;
    fileExists = false;
  }

  // 2. 檢測新檔案創建
  if (params.old_string === '' && !fileExists) {
    isNewFile = true;
  } else if (!fileExists) {
    return { error: { type: ToolErrorType.FILE_NOT_FOUND } };
  }

  // 3. LLM 編輯校正
  if (currentContent !== null) {
    const correctedEdit = await ensureCorrectEdit(
      this.resolvedPath,
      currentContent,
      params,
      this.config.getGeminiClient(),
      this.config.getBaseLlmClient(),
      abortSignal,
    );
    finalOldString = correctedEdit.params.old_string;
    finalNewString = correctedEdit.params.new_string;
    occurrences = correctedEdit.occurrences;
  }

  // 4. 錯誤檢測
  if (params.old_string === '') {
    error = { type: ToolErrorType.ATTEMPT_TO_CREATE_EXISTING_FILE };
  } else if (occurrences === 0) {
    error = { type: ToolErrorType.EDIT_NO_OCCURRENCE_FOUND };
  } else if (occurrences !== expectedReplacements) {
    error = { type: ToolErrorType.EDIT_EXPECTED_OCCURRENCE_MISMATCH };
  } else if (finalOldString === finalNewString) {
    error = { type: ToolErrorType.EDIT_NO_CHANGE };
  }

  // 5. 應用替換
  const newContent = !error
    ? applyReplacement(currentContent, finalOldString, finalNewString, isNewFile)
    : (currentContent ?? '');

  return { currentContent, newContent, occurrences, error, isNewFile };
}
```

#### 安全替換函數

```typescript
export function applyReplacement(
  currentContent: string | null,
  oldString: string,
  newString: string,
  isNewFile: boolean,
): string {
  if (isNewFile) return newString;
  if (currentContent === null) return oldString === '' ? newString : '';
  if (oldString === '' && !isNewFile) return currentContent;

  // 使用安全的字面量替換（處理 $ 符號）
  return safeLiteralReplace(currentContent, oldString, newString);
}
```

---

## 4. 搜尋工具

### 4.1 Glob 工具

**檔案位置**: `/packages/core/src/tools/glob.ts` (364 行)

#### 參數定義

```typescript
export interface GlobToolParams {
  pattern: string;               // Glob 模式 (e.g., "**/*.ts")
  dir_path?: string;             // 搜尋目錄
  case_sensitive?: boolean;      // 大小寫敏感，預設 false
  respect_git_ignore?: boolean;  // 尊重 .gitignore，預設 true
  respect_gemini_ignore?: boolean; // 尊重 .geminiignore，預設 true
}
```

#### 檔案排序演算法

```typescript
export function sortFileEntries(
  entries: GlobPath[],
  nowTimestamp: number,
  recencyThresholdMs: number,  // 24 小時
): GlobPath[] {
  const sortedEntries = [...entries];
  sortedEntries.sort((a, b) => {
    const mtimeA = a.mtimeMs ?? 0;
    const mtimeB = b.mtimeMs ?? 0;
    const aIsRecent = nowTimestamp - mtimeA < recencyThresholdMs;
    const bIsRecent = nowTimestamp - mtimeB < recencyThresholdMs;

    // 最近修改的檔案優先（按時間倒序）
    if (aIsRecent && bIsRecent) {
      return mtimeB - mtimeA;
    } else if (aIsRecent) {
      return -1;
    } else if (bIsRecent) {
      return 1;
    } else {
      // 較舊的檔案按字母順序
      return a.fullpath().localeCompare(b.fullpath());
    }
  });
  return sortedEntries;
}
```

#### 執行流程

```typescript
async execute(signal: AbortSignal): Promise<ToolResult> {
  const workspaceContext = this.config.getWorkspaceContext();

  // 確定搜尋目錄
  let searchDirectories: readonly string[];
  if (this.params.dir_path) {
    const searchDirAbsolute = path.resolve(targetDir, this.params.dir_path);
    if (!workspaceContext.isPathWithinWorkspace(searchDirAbsolute)) {
      return { error: { type: ToolErrorType.PATH_NOT_IN_WORKSPACE } };
    }
    searchDirectories = [searchDirAbsolute];
  } else {
    searchDirectories = workspaceContext.getDirectories();
  }

  // 執行 glob 搜尋
  const allEntries: GlobPath[] = [];
  for (const searchDir of searchDirectories) {
    const entries = await glob(pattern, {
      cwd: searchDir,
      withFileTypes: true,
      nodir: true,
      stat: true,
      nocase: !this.params.case_sensitive,
      dot: true,
      ignore: this.config.getFileExclusions().getGlobExcludes(),
      follow: false,
      signal,
    });
    allEntries.push(...entries);
  }

  // 過濾並排序
  const { filteredPaths, ignoredCount } = fileDiscovery.filterFilesWithReport(
    relativePaths, filterOptions
  );

  const sortedEntries = sortFileEntries(filteredEntries, Date.now(), 24*60*60*1000);

  return {
    llmContent: `Found ${fileCount} file(s) matching "${pattern}"...\n${fileList}`,
    returnDisplay: `Found ${fileCount} matching file(s)`,
  };
}
```

### 4.2 Grep 工具

**檔案位置**: `/packages/core/src/tools/grep.ts` (690 行)

#### 參數定義

```typescript
export interface GrepToolParams {
  pattern: string;      // 正規表達式模式
  dir_path?: string;    // 搜尋目錄
  include?: string;     // 檔案過濾器 (e.g., "*.js", "*.{ts,tsx}")
}
```

#### 三層搜尋策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    Grep 搜尋策略                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
┌─────────────────┐           ┌─────────────────────┐
│ Strategy 1:     │──失敗────▶│ Strategy 2:         │
│ git grep        │           │ System grep         │
│ (優先)          │           │ (回退)              │
└─────────────────┘           └──────────┬──────────┘
         │                               │
         │ 成功                          │ 失敗
         ▼                               ▼
┌─────────────────┐           ┌─────────────────────┐
│ 返回結果        │           │ Strategy 3:         │
│                 │           │ JavaScript Fallback │
└─────────────────┘           │ (最終回退)          │
                              └─────────────────────┘
```

#### Strategy 1: git grep

```typescript
if (gitAvailable) {
  const gitArgs = [
    'grep',
    '--untracked',  // 包含未追蹤檔案
    '-n',           // 顯示行號
    '-E',           // 擴展正規表達式
    '--ignore-case',
    pattern,
  ];
  if (include) gitArgs.push('--', include);

  const output = await spawn('git', gitArgs, { cwd: absolutePath });
  return this.parseGrepOutput(output, absolutePath);
}
```

#### Strategy 2: System grep

```typescript
if (grepAvailable) {
  const grepArgs = ['-r', '-n', '-H', '-E', '-I'];

  // 添加排除目錄
  commonExcludes.forEach((dir) => grepArgs.push(`--exclude-dir=${dir}`));
  if (include) grepArgs.push(`--include=${include}`);
  grepArgs.push(pattern, '.');

  const output = await spawn('grep', grepArgs, { cwd: absolutePath });
  return this.parseGrepOutput(output, absolutePath);
}
```

#### Strategy 3: JavaScript Fallback

```typescript
const filesStream = globStream(globPattern, {
  cwd: absolutePath,
  dot: true,
  ignore: ignorePatterns,
  absolute: true,
  nodir: true,
  signal: options.signal,
});

const regex = new RegExp(pattern, 'i');
const allMatches: GrepMatch[] = [];

for await (const filePath of filesStream) {
  const content = await fsPromises.readFile(filePath, 'utf8');
  const lines = content.split(/\r?\n/);
  lines.forEach((line, index) => {
    if (regex.test(line)) {
      allMatches.push({
        filePath: path.relative(absolutePath, filePath),
        lineNumber: index + 1,
        line,
      });
    }
  });
}
```

### 4.3 RipGrep 工具

**檔案位置**: `/packages/core/src/tools/ripGrep.ts` (603 行)

#### 優勢特性

| 特性 | 說明 |
|------|------|
| 效能 | 使用原生 Rust 編譯的 ripgrep |
| 自動下載 | 首次使用時自動下載 |
| 結果限制 | 預設最大 20,000 匹配 |
| 並行處理 | 4 執行緒並行搜尋 |
| 智慧忽略 | 原生支援 .gitignore |

#### 參數定義

```typescript
export interface RipGrepToolParams {
  pattern: string;           // 搜尋模式
  dir_path?: string;         // 搜尋目錄
  include?: string;          // Glob 過濾器
  case_sensitive?: boolean;  // 大小寫敏感
  fixed_strings?: boolean;   // 字面量模式
  context?: number;          // 上下文行數
  after?: number;            // 匹配後行數
  before?: number;           // 匹配前行數
  no_ignore?: boolean;       // 忽略 .gitignore
}
```

#### ripgrep 二進位管理

```typescript
async function ensureRipgrepAvailable(): Promise<string | null> {
  const existingPath = await resolveExistingRgPath();
  if (existingPath) return existingPath;

  if (!ripgrepAcquisitionPromise) {
    ripgrepAcquisitionPromise = (async () => {
      try {
        await downloadRipGrep(Storage.getGlobalBinDir());
        return await resolveExistingRgPath();
      } finally {
        ripgrepAcquisitionPromise = null;
      }
    })();
  }
  return ripgrepAcquisitionPromise;
}
```

#### JSON 輸出解析

```typescript
private parseRipgrepJsonOutput(output: string, basePath: string): GrepMatch[] {
  const results: GrepMatch[] = [];
  const lines = output.trim().split('\n');

  for (const line of lines) {
    try {
      const json = JSON.parse(line);
      if (json.type === 'match') {
        const match = json.data;
        if (match.path?.text && match.lines?.text) {
          results.push({
            filePath: path.relative(basePath, match.path.text),
            lineNumber: match.line_number,
            line: match.lines.text.trimEnd(),
          });
        }
      }
    } catch (error) {
      debugLogger.warn(`Failed to parse ripgrep JSON line: ${line}`);
    }
  }
  return results;
}
```

---

## 5. Shell 工具

**檔案位置**: `/packages/core/src/tools/shell.ts` (543 行)

### 5.1 參數定義

```typescript
export interface ShellToolParams {
  command: string;       // Shell 命令
  description?: string;  // 人類可讀描述
  dir_path?: string;     // 工作目錄
}
```

### 5.2 平台適配

```typescript
function getShellToolDescription(): string {
  if (os.platform() === 'win32') {
    return `Executes command as \`powershell.exe -NoProfile -Command <command>\``;
  } else {
    return `Executes command as \`bash -c <command>\``;
  }
}
```

### 5.3 執行策略

```typescript
async execute(
  signal: AbortSignal,
  updateOutput?: (output: string | AnsiOutput) => void,
  shellExecutionConfig?: ShellExecutionConfig,
  setPidCallback?: (pid: number) => void,
): Promise<ToolResult> {
  const strippedCommand = stripShellWrapper(this.params.command);

  // 構建命令（非 Windows 平台添加 pgrep 以獲取背景 PID）
  const commandToExecute = isWindows
    ? strippedCommand
    : `{ ${strippedCommand}; }; __code=$?; pgrep -g 0 >${tempFilePath}; exit $__code;`;

  // 超時管理
  const timeoutMs = this.config.getShellToolInactivityTimeout();
  const resetTimeout = () => {
    if (timeoutMs <= 0) return;
    if (timeoutTimer) clearTimeout(timeoutTimer);
    timeoutTimer = setTimeout(() => timeoutController.abort(), timeoutMs);
  };

  // 執行命令
  const { result: resultPromise, pid } = await ShellExecutionService.execute(
    commandToExecute,
    cwd,
    (event: ShellOutputEvent) => {
      resetTimeout();  // 有輸出時重置超時
      if (!updateOutput) return;

      switch (event.type) {
        case 'data':
          cumulativeOutput = event.chunk;
          break;
        case 'binary_detected':
          cumulativeOutput = '[Binary output detected...]';
          break;
        case 'binary_progress':
          cumulativeOutput = `[Receiving binary... ${formatMemoryUsage(event.bytesReceived)}]`;
          break;
      }
      updateOutput(cumulativeOutput);
    },
    combinedController.signal,
    this.config.getEnableInteractiveShell(),
    shellExecutionConfig,
  );

  // 處理結果
  const result = await resultPromise;

  // 收集背景 PID
  const backgroundPIDs: number[] = [];
  if (!isWindows && fs.existsSync(tempFilePath)) {
    const pgrepLines = fs.readFileSync(tempFilePath, 'utf8').split(EOL);
    for (const line of pgrepLines) {
      const pid = Number(line);
      if (pid !== result.pid) backgroundPIDs.push(pid);
    }
  }

  // 構建輸出
  let llmContent = [
    `Command: ${this.params.command}`,
    `Directory: ${this.params.dir_path || '(root)'}`,
    `Output: ${result.output || '(empty)'}`,
    `Error: ${result.error?.message ?? '(none)'}`,
    `Exit Code: ${result.exitCode ?? '(none)'}`,
    `Signal: ${result.signal ?? '(none)'}`,
    `Background PIDs: ${backgroundPIDs.length ? backgroundPIDs.join(', ') : '(none)'}`,
    `Process Group PGID: ${result.pid ?? '(none)'}`,
  ].join('\n');

  return { llmContent, returnDisplay: result.output || '', ...executionError };
}
```

### 5.4 安全機制

#### 命令驗證

```typescript
protected override validateToolParamValues(params: ShellToolParams): string | null {
  if (!params.command.trim()) {
    return 'Command cannot be empty.';
  }

  // 檢查命令是否被允許
  const commandCheck = isCommandAllowed(params.command, this.config);
  if (!commandCheck.allowed) {
    return commandCheck.reason;
  }

  // 提取命令根以獲取使用者權限
  if (getCommandRoots(params.command).length === 0) {
    return 'Could not identify command root to obtain permission from user.';
  }

  // 驗證工作目錄
  if (params.dir_path) {
    const resolvedPath = path.resolve(this.config.getTargetDir(), params.dir_path);
    if (!workspaceContext.isPathWithinWorkspace(resolvedPath)) {
      return `Directory is not within workspace directories.`;
    }
  }

  return null;
}
```

#### Shell Wrapper 剝離

```typescript
// 從 shell-utils.ts
export function stripShellWrapper(command: string): string {
  // 移除 bash -c "..." 或 sh -c "..." 包裝
  const wrapperPatterns = [
    /^(bash|sh)\s+-c\s+(['"])(.+)\2$/,
    /^(bash|sh)\s+-c\s+(.+)$/,
  ];

  for (const pattern of wrapperPatterns) {
    const match = command.match(pattern);
    if (match) return match[3] || match[2];
  }
  return command;
}

export function getCommandRoots(command: string): string[] {
  // 提取命令的根命令（用於權限檢查）
  // 例如: "npm install && npm test" -> ["npm"]
  // ...
}
```

#### 確認流程

```typescript
protected override async getConfirmationDetails(
  _abortSignal: AbortSignal,
): Promise<ToolCallConfirmationDetails | false> {
  const command = stripShellWrapper(this.params.command);
  const rootCommands = [...new Set(getCommandRoots(command))];

  // 非互動模式檢查
  if (!this.config.isInteractive() &&
      this.config.getApprovalMode() !== ApprovalMode.YOLO) {
    if (this.isInvocationAllowlisted(command)) {
      return false;
    }
    throw new Error(`Command is not in the list of allowed tools for non-interactive mode.`);
  }

  // 過濾已允許的命令
  const commandsToConfirm = rootCommands.filter(
    (cmd) => !this.allowlist.has(cmd)
  );

  if (commandsToConfirm.length === 0) return false;

  return {
    type: 'exec',
    title: 'Confirm Shell Command',
    command: this.params.command,
    rootCommand: commandsToConfirm.join(', '),
    onConfirm: async (outcome) => {
      if (outcome === ToolConfirmationOutcome.ProceedAlways) {
        commandsToConfirm.forEach((cmd) => this.allowlist.add(cmd));
      }
      await this.publishPolicyUpdate(outcome);
    },
  };
}
```

---

## 6. Web 工具

### 6.1 WebFetch 工具

**檔案位置**: `/packages/core/src/tools/web-fetch.ts` (468 行)

#### 參數定義

```typescript
export interface WebFetchToolParams {
  prompt: string;  // 包含 URL 和處理指令的提示
}
```

#### URL 解析

```typescript
export function parsePrompt(text: string): {
  validUrls: string[];
  errors: string[];
} {
  const tokens = text.split(/\s+/);
  const validUrls: string[] = [];
  const errors: string[] = [];

  for (const token of tokens) {
    if (token.includes('://')) {
      try {
        const url = new URL(token);
        if (['http:', 'https:'].includes(url.protocol)) {
          validUrls.push(url.href);
        } else {
          errors.push(`Unsupported protocol: "${token}"`);
        }
      } catch (_) {
        errors.push(`Malformed URL: "${token}"`);
      }
    }
  }
  return { validUrls, errors };
}
```

#### 執行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    WebFetch 執行流程                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 解析 URL 並檢查私有 IP                                        │
│    └── 私有 IP? → executeFallback()                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 主要路徑: Gemini API 處理                                     │
│    ├── geminiClient.generateContent({ model: 'web-fetch' })     │
│    ├── 獲取 grounding metadata                                   │
│    └── 處理 URL 檢索狀態                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
               ┌─────────┴─────────┐
               │                   │
          成功  │                   │ 失敗
               ▼                   ▼
┌─────────────────┐     ┌─────────────────────┐
│ 格式化回應      │     │ Fallback: 直接獲取  │
│ 添加來源引用    │     │ ├── fetchWithTimeout │
│ 返回結果       │      │ ├── HTML → 文字轉換  │
└─────────────────┘     │ └── LLM 處理內容    │
                        └─────────────────────┘
```

#### Fallback 機制

```typescript
private async executeFallback(signal: AbortSignal): Promise<ToolResult> {
  const { validUrls: urls } = parsePrompt(this.params.prompt);
  let url = urls[0];

  // GitHub blob → raw 轉換
  if (url.includes('github.com') && url.includes('/blob/')) {
    url = url
      .replace('github.com', 'raw.githubusercontent.com')
      .replace('/blob/', '/');
  }

  const response = await retryWithBackoff(
    async () => {
      const res = await fetchWithTimeout(url, URL_FETCH_TIMEOUT_MS);
      if (!res.ok) throw new Error(`Request failed: ${res.status}`);
      return res;
    },
    { retryFetchErrors: this.config.getRetryFetchErrors() },
  );

  const rawContent = await response.text();
  const contentType = response.headers.get('content-type') || '';

  // HTML 轉文字
  let textContent: string;
  if (contentType.includes('text/html') || contentType === '') {
    textContent = convert(rawContent, {
      wordwrap: false,
      selectors: [
        { selector: 'a', options: { ignoreHref: true } },
        { selector: 'img', format: 'skip' },
      ],
    });
  } else {
    textContent = rawContent;
  }

  textContent = textContent.substring(0, MAX_CONTENT_LENGTH);  // 100,000 字元限制

  // 使用 LLM 處理
  const result = await geminiClient.generateContent(
    { model: 'web-fetch-fallback' },
    [{ role: 'user', parts: [{ text: fallbackPrompt }] }],
    signal,
  );

  return { llmContent: getResponseText(result), returnDisplay: `Content processed.` };
}
```

### 6.2 WebSearch 工具

**檔案位置**: `/packages/core/src/tools/web-search.ts` (247 行)

#### 參數定義

```typescript
export interface WebSearchToolParams {
  query: string;  // 搜尋查詢
}

export interface WebSearchToolResult extends ToolResult {
  sources?: GroundingChunkItem[];  // 搜尋來源
}
```

#### 執行流程

```typescript
async execute(signal: AbortSignal): Promise<WebSearchToolResult> {
  const response = await geminiClient.generateContent(
    { model: 'web-search' },
    [{ role: 'user', parts: [{ text: this.params.query }] }],
    signal,
  );

  const responseText = getResponseText(response);
  const groundingMetadata = response.candidates?.[0]?.groundingMetadata;
  const sources = groundingMetadata?.groundingChunks;
  const groundingSupports = groundingMetadata?.groundingSupports;

  if (!responseText?.trim()) {
    return {
      llmContent: `No results found for: "${this.params.query}"`,
      returnDisplay: 'No information found.',
    };
  }

  let modifiedResponseText = responseText;

  // 處理來源引用
  if (sources?.length > 0) {
    const sourceListFormatted = sources.map((source, index) =>
      `[${index + 1}] ${source.web?.title || 'Untitled'} (${source.web?.uri || 'No URI'})`
    );

    // 插入引用標記
    if (groundingSupports?.length > 0) {
      const insertions = groundingSupports.map(support => ({
        index: support.segment?.endIndex,
        marker: support.groundingChunkIndices?.map(i => `[${i + 1}]`).join(''),
      })).sort((a, b) => b.index - a.index);

      // UTF-8 字節位置處理
      const encoder = new TextEncoder();
      const responseBytes = encoder.encode(modifiedResponseText);
      // ... 插入標記邏輯
    }

    modifiedResponseText += '\n\nSources:\n' + sourceListFormatted.join('\n');
  }

  return {
    llmContent: `Web search results for "${this.params.query}":\n\n${modifiedResponseText}`,
    returnDisplay: `Search results returned.`,
    sources,
  };
}
```

---

## 7. MCP 工具系統

### 7.1 MCPClientAdapter

**檔案位置**: `/packages/core/src/tools/mcp-client.ts` (1,847 行)

#### 伺服器狀態管理

```typescript
export enum MCPServerStatus {
  DISCONNECTED = 'disconnected',   // 斷開或錯誤
  DISCONNECTING = 'disconnecting', // 正在斷開
  CONNECTING = 'connecting',       // 正在連接
  CONNECTED = 'connected',         // 已連接就緒
}

export enum MCPDiscoveryState {
  NOT_STARTED = 'not_started',
  IN_PROGRESS = 'in_progress',
  COMPLETED = 'completed',
}
```

#### McpClient 類別

```typescript
export class McpClient {
  private client: Client | undefined;
  private transport: Transport | undefined;
  private status: MCPServerStatus = MCPServerStatus.DISCONNECTED;
  private isRefreshingTools: boolean = false;
  private pendingToolRefresh: boolean = false;

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

  async connect(): Promise<void>;
  async discover(cliConfig: Config): Promise<void>;
  async disconnect(): Promise<void>;
  async readResource(uri: string): Promise<ReadResourceResult>;
}
```

### 7.2 傳輸層支援

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP 傳輸層架構                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ StdioClient     │  │ SSEClient       │  │ StreamableHTTP  │  │
│  │ Transport       │  │ Transport       │  │ ClientTransport │  │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤  │
│  │ 本地程序        │  │ Server-Sent     │  │ Streamable HTTP │  │
│  │ stdin/stdout    │  │ Events (SSE)    │  │ JSON-RPC        │  │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤  │
│  │ 配置:          │  │ 配置:           │  │ 配置:           │  │
│  │ - command      │  │ - url           │  │ - httpUrl       │  │
│  │ - args         │  │ - type: 'sse'   │  │ - type: 'http'  │  │
│  │ - env          │  │                 │  │                 │  │
│  │ - cwd          │  │                 │  │                 │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 傳輸創建邏輯

```typescript
export async function createTransport(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  debugMode: boolean,
  sanitizationConfig: EnvironmentSanitizationConfig,
): Promise<Transport> {
  // 網路傳輸
  if (mcpServerConfig.httpUrl || mcpServerConfig.url) {
    const authProvider = createAuthProvider(mcpServerConfig);
    const headers = await authProvider?.getRequestHeaders?.() ?? {};

    // OAuth 令牌處理
    if (!authProvider) {
      const accessToken = await getStoredOAuthToken(mcpServerName);
      if (accessToken) headers['Authorization'] = `Bearer ${accessToken}`;
    }

    return createUrlTransport(mcpServerName, mcpServerConfig, {
      requestInit: { headers },
      authProvider,
    });
  }

  // Stdio 傳輸
  if (mcpServerConfig.command) {
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

    if (debugMode) {
      transport.stderr!.on('data', (data) => {
        debugLogger.debug(`[MCP STDERR (${mcpServerName})]:`, data.toString());
      });
    }
    return transport;
  }

  throw new Error('Invalid configuration: missing URL or command.');
}
```

### 7.3 DiscoveredMCPTool

**檔案位置**: `/packages/core/src/tools/mcp-tool.ts` (447 行)

```typescript
export class DiscoveredMCPTool extends BaseDeclarativeTool<ToolParams, ToolResult> {
  constructor(
    private readonly mcpTool: CallableTool,
    readonly serverName: string,
    readonly serverToolName: string,
    description: string,
    parameterSchema: unknown,
    readonly trust?: boolean,
    nameOverride?: string,
    private readonly cliConfig?: Config,
    extensionName?: string,
    extensionId?: string,
    messageBus?: MessageBus,
  ) {
    super(
      nameOverride ?? generateValidName(serverToolName),
      `${serverToolName} (${serverName} MCP Server)`,
      description,
      Kind.Other,
      parameterSchema,
      true, false,
      messageBus,
      extensionName,
      extensionId,
    );
  }

  getFullyQualifiedPrefix(): string {
    return `${this.serverName}__`;
  }

  asFullyQualifiedTool(): DiscoveredMCPTool {
    return new DiscoveredMCPTool(
      this.mcpTool, this.serverName, this.serverToolName,
      this.description, this.parameterSchema, this.trust,
      `${this.getFullyQualifiedPrefix()}${this.serverToolName}`,
      this.cliConfig, this.extensionName, this.extensionId, this.messageBus,
    );
  }
}
```

#### MCP 內容轉換

```typescript
type McpContentBlock =
  | { type: 'text'; text: string }
  | { type: 'image' | 'audio'; mimeType: string; data: string }
  | { type: 'resource'; resource: { text?: string; blob?: string; mimeType?: string } }
  | { type: 'resource_link'; uri: string; title?: string; name?: string };

function transformMcpContentToParts(sdkResponse: Part[]): Part[] {
  const funcResponse = sdkResponse?.[0]?.functionResponse;
  const mcpContent = funcResponse?.response?.['content'] as McpContentBlock[];

  return mcpContent.flatMap((block): Part | Part[] | null => {
    switch (block.type) {
      case 'text':
        return { text: block.text };
      case 'image':
      case 'audio':
        return [
          { text: `[Tool provided ${block.type} with mime-type: ${block.mimeType}]` },
          { inlineData: { mimeType: block.mimeType, data: block.data } },
        ];
      case 'resource':
        if (block.resource?.text) return { text: block.resource.text };
        if (block.resource?.blob) {
          return [
            { text: `[Embedded resource: ${block.resource.mimeType}]` },
            { inlineData: { mimeType: block.resource.mimeType, data: block.resource.blob } },
          ];
        }
        return null;
      case 'resource_link':
        return { text: `Resource Link: ${block.title || block.name} at ${block.uri}` };
      default:
        return null;
    }
  }).filter(Boolean);
}
```

---

## 8. 記憶工具

**檔案位置**: `/packages/core/src/tools/memoryTool.ts` (394 行)

### 8.1 參數定義

```typescript
interface SaveMemoryParams {
  fact: string;                    // 要記住的事實
  modified_by_user?: boolean;      // 使用者修改標記
  modified_content?: string;       // 修改後的內容
}
```

### 8.2 記憶檔案管理

```typescript
export const DEFAULT_CONTEXT_FILENAME = 'GEMINI.md';
export const MEMORY_SECTION_HEADER = '## Gemini Added Memories';

let currentGeminiMdFilename: string | string[] = DEFAULT_CONTEXT_FILENAME;

export function getGlobalMemoryFilePath(): string {
  return path.join(Storage.getGlobalGeminiDir(), getCurrentGeminiMdFilename());
}
```

### 8.3 內容計算

```typescript
function computeNewContent(currentContent: string, fact: string): string {
  let processedText = fact.trim();
  processedText = processedText.replace(/^(-+\s*)+/, '').trim();
  const newMemoryItem = `- ${processedText}`;

  const headerIndex = currentContent.indexOf(MEMORY_SECTION_HEADER);

  if (headerIndex === -1) {
    // 添加新區塊
    const separator = ensureNewlineSeparation(currentContent);
    return currentContent + `${separator}${MEMORY_SECTION_HEADER}\n${newMemoryItem}\n`;
  } else {
    // 在現有區塊中添加
    const startOfSectionContent = headerIndex + MEMORY_SECTION_HEADER.length;
    let endOfSectionIndex = currentContent.indexOf('\n## ', startOfSectionContent);
    if (endOfSectionIndex === -1) endOfSectionIndex = currentContent.length;

    const beforeSection = currentContent.substring(0, startOfSectionContent).trimEnd();
    let sectionContent = currentContent.substring(startOfSectionContent, endOfSectionIndex).trimEnd();
    const afterSection = currentContent.substring(endOfSectionIndex);

    sectionContent += `\n${newMemoryItem}`;
    return `${beforeSection}\n${sectionContent.trimStart()}\n${afterSection}`.trimEnd() + '\n';
  }
}
```

### 8.4 使用準則

```
┌─────────────────────────────────────────────────────────────────┐
│                    記憶工具使用準則                              │
├────────────────────────────┬────────────────────────────────────┤
│         使用時機           │          不使用時機                │
├────────────────────────────┼────────────────────────────────────┤
│ ✓ 使用者明確要求記住       │ ✗ 僅限當前會話的上下文             │
│ ✓ 關於偏好/環境的重要事實  │ ✗ 長/複雜/冗長的文字               │
│ ✓ 自包含、清晰的陳述       │ ✗ 不確定重要性的資訊               │
│ ✓ 個人化助手行為           │ ✗ 臨時性或一次性的資訊             │
└────────────────────────────┴────────────────────────────────────┘
```

---

## 9. 智慧編輯工具 (SmartEdit)

**檔案位置**: `/packages/core/src/tools/smart-edit.ts` (1,014 行)

### 9.1 參數定義

```typescript
export interface EditToolParams {
  file_path: string;              // 檔案路徑
  old_string: string;             // 要替換的文字
  new_string: string;             // 替換後的文字
  expected_replacements?: number; // 預期替換次數
  instruction: string;            // 變更指令（SmartEdit 特有）
  modified_by_user?: boolean;
  ai_proposed_string?: string;
}
```

### 9.2 三種替換策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    SmartEdit 替換策略                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Strategy 1  │  │ Strategy 2  │  │ Strategy 3  │
│ 精確替換    │  │ 彈性替換    │  │ 正規替換    │
│ (Exact)     │  │ (Flexible)  │  │ (Regex)     │
├─────────────┤  ├─────────────┤  ├─────────────┤
│ 完全匹配    │  │ 忽略空白    │  │ Token 化    │
│ 字面量替換  │  │ 滑動視窗    │  │ 靈活匹配    │
└─────────────┘  └─────────────┘  └─────────────┘
         │               │               │
         └───────────────┴───────────────┘
                         │
                    全部失敗
                         │
                         ▼
              ┌─────────────────┐
              │ LLM 自我校正    │
              │ (Self-Correction)│
              └─────────────────┘
```

#### Strategy 1: 精確替換

```typescript
async function calculateExactReplacement(
  context: ReplacementContext,
): Promise<ReplacementResult | null> {
  const { currentContent, params } = context;

  const normalizedCode = currentContent;
  const normalizedSearch = params.old_string.replace(/\r\n/g, '\n');
  const normalizedReplace = params.new_string.replace(/\r\n/g, '\n');

  const exactOccurrences = normalizedCode.split(normalizedSearch).length - 1;

  if (exactOccurrences > 0) {
    let modifiedCode = safeLiteralReplace(normalizedCode, normalizedSearch, normalizedReplace);
    modifiedCode = restoreTrailingNewline(currentContent, modifiedCode);
    return {
      newContent: modifiedCode,
      occurrences: exactOccurrences,
      finalOldString: normalizedSearch,
      finalNewString: normalizedReplace,
    };
  }
  return null;
}
```

#### Strategy 2: 彈性替換

```typescript
async function calculateFlexibleReplacement(
  context: ReplacementContext,
): Promise<ReplacementResult | null> {
  const { currentContent, params } = context;

  const sourceLines = normalizedCode.match(/.*(?:\n|$)/g)?.slice(0, -1) ?? [];
  const searchLinesStripped = normalizedSearch.split('\n').map(line => line.trim());
  const replaceLines = normalizedReplace.split('\n');

  let flexibleOccurrences = 0;
  let i = 0;

  while (i <= sourceLines.length - searchLinesStripped.length) {
    const window = sourceLines.slice(i, i + searchLinesStripped.length);
    const windowStripped = window.map(line => line.trim());

    // 比較去除空白後的內容
    const isMatch = windowStripped.every(
      (line, index) => line === searchLinesStripped[index]
    );

    if (isMatch) {
      flexibleOccurrences++;
      // 保持原始縮進
      const indentationMatch = window[0].match(/^(\s*)/);
      const indentation = indentationMatch ? indentationMatch[1] : '';
      const newBlockWithIndent = replaceLines.map(line => `${indentation}${line}`);
      sourceLines.splice(i, searchLinesStripped.length, newBlockWithIndent.join('\n'));
      i += replaceLines.length;
    } else {
      i++;
    }
  }

  if (flexibleOccurrences > 0) {
    return {
      newContent: restoreTrailingNewline(currentContent, sourceLines.join('')),
      occurrences: flexibleOccurrences,
      finalOldString: normalizedSearch,
      finalNewString: normalizedReplace,
    };
  }
  return null;
}
```

#### Strategy 3: 正規表達式替換

```typescript
async function calculateRegexReplacement(
  context: ReplacementContext,
): Promise<ReplacementResult | null> {
  const { currentContent, params } = context;

  // Token 化處理
  const delimiters = ['(', ')', ':', '[', ']', '{', '}', '>', '<', '='];
  let processedString = normalizedSearch;
  for (const delim of delimiters) {
    processedString = processedString.split(delim).join(` ${delim} `);
  }

  const tokens = processedString.split(/\s+/).filter(Boolean);
  if (tokens.length === 0) return null;

  const escapedTokens = tokens.map(escapeRegex);
  const pattern = escapedTokens.join('\\s*');

  // 捕獲縮進
  const finalPattern = `^(\\s*)${pattern}`;
  const flexibleRegex = new RegExp(finalPattern, 'm');

  const match = flexibleRegex.exec(currentContent);
  if (!match) return null;

  const indentation = match[1] || '';
  const newBlockWithIndent = normalizedReplace.split('\n')
    .map(line => `${indentation}${line}`)
    .join('\n');

  const modifiedCode = currentContent.replace(flexibleRegex, newBlockWithIndent);

  return {
    newContent: restoreTrailingNewline(currentContent, modifiedCode),
    occurrences: 1,
    finalOldString: normalizedSearch,
    finalNewString: normalizedReplace,
  };
}
```

### 9.3 LLM 自我校正

```typescript
private async attemptSelfCorrection(
  params: EditToolParams,
  currentContent: string,
  initialError: { display: string; raw: string; type: ToolErrorType },
  abortSignal: AbortSignal,
  originalLineEnding: '\r\n' | '\n',
): Promise<CalculatedEdit> {
  // 檢查檔案是否在讀取後被修改
  const initialContentHash = hashContent(currentContent);
  const onDiskContent = await this.config.getFileSystemService()
    .readTextFile(params.file_path);
  const onDiskContentHash = hashContent(onDiskContent.replace(/\r\n/g, '\n'));

  let errorForLlmEditFixer = initialError.raw;
  let contentForLlmEditFixer = currentContent;

  if (initialContentHash !== onDiskContentHash) {
    contentForLlmEditFixer = onDiskContent.replace(/\r\n/g, '\n');
    errorForLlmEditFixer = `The file has been modified. Use the latest content.`;
  }

  // 調用 LLM 進行修復
  const fixedEdit = await FixLLMEditWithInstruction(
    params.instruction,
    params.old_string,
    params.new_string,
    errorForLlmEditFixer,
    contentForLlmEditFixer,
    this.config.getBaseLlmClient(),
    abortSignal,
  );

  if (fixedEdit === null) {
    // 超時，返回原始錯誤
    return { ...originalResult, error: initialError };
  }

  if (fixedEdit.noChangesRequired) {
    return {
      error: {
        type: ToolErrorType.EDIT_NO_CHANGE_LLM_JUDGEMENT,
        raw: `LLM determined no changes necessary: ${fixedEdit.explanation}`,
      },
    };
  }

  // 使用修復後的參數重試
  const secondAttemptResult = await calculateReplacement(this.config, {
    params: { ...params, old_string: fixedEdit.search, new_string: fixedEdit.replace },
    currentContent: contentForLlmEditFixer,
    abortSignal,
  });

  const secondError = getErrorReplaceResult(...);

  if (secondError) {
    logSmartEditCorrectionEvent(this.config, new SmartEditCorrectionEvent('failure'));
    return { ...originalResult, error: initialError };
  }

  logSmartEditCorrectionEvent(this.config, new SmartEditCorrectionEvent('success'));
  return { ...secondAttemptResult, error: undefined };
}
```

---

## 10. 錯誤處理系統

**檔案位置**: `/packages/core/src/tools/tool-error.ts` (104 行)

### 10.1 ToolErrorType 完整列舉

```typescript
export enum ToolErrorType {
  // 一般錯誤
  INVALID_TOOL_PARAMS = 'invalid_tool_params',
  UNKNOWN = 'unknown',
  UNHANDLED_EXCEPTION = 'unhandled_exception',
  TOOL_NOT_REGISTERED = 'tool_not_registered',
  EXECUTION_FAILED = 'execution_failed',

  // 檔案系統錯誤
  FILE_NOT_FOUND = 'file_not_found',
  FILE_WRITE_FAILURE = 'file_write_failure',
  READ_CONTENT_FAILURE = 'read_content_failure',
  ATTEMPT_TO_CREATE_EXISTING_FILE = 'attempt_to_create_existing_file',
  FILE_TOO_LARGE = 'file_too_large',
  PERMISSION_DENIED = 'permission_denied',
  NO_SPACE_LEFT = 'no_space_left',
  TARGET_IS_DIRECTORY = 'target_is_directory',
  PATH_NOT_IN_WORKSPACE = 'path_not_in_workspace',
  SEARCH_PATH_NOT_FOUND = 'search_path_not_found',
  SEARCH_PATH_NOT_A_DIRECTORY = 'search_path_not_a_directory',

  // 編輯特定錯誤
  EDIT_PREPARATION_FAILURE = 'edit_preparation_failure',
  EDIT_NO_OCCURRENCE_FOUND = 'edit_no_occurrence_found',
  EDIT_EXPECTED_OCCURRENCE_MISMATCH = 'edit_expected_occurrence_mismatch',
  EDIT_NO_CHANGE = 'edit_no_change',
  EDIT_NO_CHANGE_LLM_JUDGEMENT = 'edit_no_change_llm_judgement',

  // 搜尋工具錯誤
  GLOB_EXECUTION_ERROR = 'glob_execution_error',
  GREP_EXECUTION_ERROR = 'grep_execution_error',
  LS_EXECUTION_ERROR = 'ls_execution_error',
  PATH_IS_NOT_A_DIRECTORY = 'path_is_not_a_directory',

  // MCP 錯誤
  MCP_TOOL_ERROR = 'mcp_tool_error',

  // 其他工具錯誤
  MEMORY_TOOL_EXECUTION_ERROR = 'memory_tool_execution_error',
  READ_MANY_FILES_SEARCH_ERROR = 'read_many_files_search_error',
  SHELL_EXECUTE_ERROR = 'shell_execute_error',
  DISCOVERED_TOOL_EXECUTION_ERROR = 'discovered_tool_execution_error',

  // Web 工具錯誤
  WEB_FETCH_NO_URL_IN_PROMPT = 'web_fetch_no_url_in_prompt',
  WEB_FETCH_FALLBACK_FAILED = 'web_fetch_fallback_failed',
  WEB_FETCH_PROCESSING_ERROR = 'web_fetch_processing_error',
  WEB_SEARCH_FAILED = 'web_search_failed',

  // Hook 錯誤
  STOP_EXECUTION = 'stop_execution',
}
```

### 10.2 致命錯誤判斷

```typescript
/**
 * 判斷錯誤是否為致命錯誤
 *
 * 致命錯誤: 系統級問題，繼續執行不太可能成功
 * - NO_SPACE_LEFT: 磁碟空間不足
 *
 * 非致命錯誤: LLM 可能可以自我恢復
 * - INVALID_TOOL_PARAMS: 可以校正參數
 * - FILE_NOT_FOUND: 可以嘗試其他檔案
 * - PATH_NOT_IN_WORKSPACE: 可以使用不同路徑
 * - PERMISSION_DENIED: 可以使用其他方法
 */
export function isFatalToolError(errorType?: string): boolean {
  if (!errorType) return false;

  const fatalErrors = new Set<string>([
    ToolErrorType.NO_SPACE_LEFT,
  ]);

  return fatalErrors.has(errorType);
}
```

### 10.3 錯誤分類表

| 錯誤類別 | 錯誤類型 | 可恢復性 | 建議處理 |
|---------|---------|---------|---------|
| 一般 | INVALID_TOOL_PARAMS | 可恢復 | LLM 重試 |
| 一般 | EXECUTION_FAILED | 可恢復 | 重試或替代方法 |
| 檔案 | FILE_NOT_FOUND | 可恢復 | 檢查路徑 |
| 檔案 | PERMISSION_DENIED | 可恢復 | 使用其他檔案 |
| 檔案 | NO_SPACE_LEFT | **致命** | 終止執行 |
| 編輯 | EDIT_NO_OCCURRENCE_FOUND | 可恢復 | 讀取檔案重試 |
| 編輯 | EDIT_NO_CHANGE_LLM_JUDGEMENT | 可恢復 | 無需操作 |
| MCP | MCP_TOOL_ERROR | 可恢復 | 檢查伺服器 |
| Web | WEB_FETCH_FALLBACK_FAILED | 可恢復 | 手動獲取 |

---

## 11. 三層驗證架構詳解

### 11.1 架構概覽

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           三層驗證架構                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        第 1 層: Schema 驗證                          │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │    │
│  │  │ JSON Schema     │  │ 類型檢查        │  │ 必填欄位            │  │    │
│  │  │ Validation      │  │ Type Checking   │  │ Required Fields     │  │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │    │
│  │                           ↓ 通過                                     │    │
│  └───────────────────────────┼─────────────────────────────────────────┘    │
│                              │                                               │
│  ┌───────────────────────────┼─────────────────────────────────────────┐    │
│  │                        第 2 層: 參數驗證                             │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │    │
│  │  │ 業務邏輯檢查    │  │ 工作區邊界      │  │ 路徑遍歷防護        │  │    │
│  │  │ Business Logic  │  │ Workspace Bound │  │ Path Traversal      │  │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │    │
│  │  ┌─────────────────┐  ┌─────────────────┐                           │    │
│  │  │ 檔案存在檢查    │  │ 工具特定約束    │                           │    │
│  │  │ File Existence  │  │ Tool-specific   │                           │    │
│  │  └─────────────────┘  └─────────────────┘                           │    │
│  │                           ↓ 通過                                     │    │
│  └───────────────────────────┼─────────────────────────────────────────┘    │
│                              │                                               │
│  ┌───────────────────────────┼─────────────────────────────────────────┐    │
│  │                        第 3 層: 執行批准                             │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │    │
│  │  │ 政策決策        │  │ 使用者確認      │  │ 政策持久化          │  │    │
│  │  │ Policy Decision │  │ User Confirm    │  │ Policy Persistence  │  │    │
│  │  │ ALLOW/DENY/ASK  │  │                 │  │                     │  │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │    │
│  │                           ↓ 批准                                     │    │
│  └───────────────────────────┼─────────────────────────────────────────┘    │
│                              │                                               │
│                              ▼                                               │
│                      ┌───────────────┐                                       │
│                      │   執行工具    │                                       │
│                      │   Execute     │                                       │
│                      └───────────────┘                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 第 1 層: Schema 驗證

```typescript
// SchemaValidator.validate() 實現
export class SchemaValidator {
  static validate(schema: unknown, params: unknown): string | null {
    // 使用 Ajv 進行 JSON Schema 驗證
    const ajv = new Ajv({ allErrors: true });
    const validate = ajv.compile(schema as JSONSchemaType<unknown>);

    if (!validate(params)) {
      return validate.errors
        ?.map(e => `${e.instancePath} ${e.message}`)
        .join('; ');
    }
    return null;
  }
}

// BaseDeclarativeTool 中的使用
override validateToolParams(params: TParams): string | null {
  // 第 1 層: JSON Schema 驗證
  const errors = SchemaValidator.validate(
    this.schema.parametersJsonSchema,
    params,
  );
  if (errors) return errors;

  // 第 2 層: 業務邏輯驗證
  return this.validateToolParamValues(params);
}
```

### 11.3 第 2 層: 參數驗證

```typescript
// 典型的 validateToolParamValues 實現
protected override validateToolParamValues(params: EditToolParams): string | null {
  // 1. 非空檢查
  if (!params.file_path) {
    return "The 'file_path' parameter must be non-empty.";
  }

  // 2. 路徑解析
  const resolvedPath = path.resolve(this.config.getTargetDir(), params.file_path);

  // 3. 工作區邊界檢查
  const workspaceContext = this.config.getWorkspaceContext();
  if (!workspaceContext.isPathWithinWorkspace(resolvedPath)) {
    const directories = workspaceContext.getDirectories();
    return `File path must be within workspace directories: ${directories.join(', ')}`;
  }

  // 4. 檔案/目錄類型檢查
  try {
    const stats = fs.statSync(resolvedPath);
    if (stats.isDirectory()) {
      return `Path is a directory, not a file: ${resolvedPath}`;
    }
  } catch (error) {
    // 檔案不存在可能是合法的（創建新檔案）
  }

  // 5. 工具特定驗證
  // ...

  return null;
}
```

### 11.4 第 3 層: 執行批准

```typescript
// BaseToolInvocation 中的政策決策
protected getMessageBusDecision(
  abortSignal: AbortSignal,
): Promise<'ALLOW' | 'DENY' | 'ASK_USER'> {
  if (!this.messageBus) {
    return Promise.resolve('ALLOW');
  }

  const correlationId = randomUUID();
  const toolCall = {
    name: this._toolName || this.constructor.name,
    args: this.params as Record<string, unknown>,
  };

  return new Promise<'ALLOW' | 'DENY' | 'ASK_USER'>((resolve) => {
    let timeoutId: NodeJS.Timeout | undefined;

    const responseHandler = (response: ToolConfirmationResponse) => {
      if (response.correlationId === correlationId) {
        cleanup();
        if (response.requiresUserConfirmation) {
          resolve('ASK_USER');
        } else if (response.confirmed) {
          resolve('ALLOW');
        } else {
          resolve('DENY');
        }
      }
    };

    // 30 秒超時
    timeoutId = setTimeout(() => {
      cleanup();
      resolve('ASK_USER');
    }, 30000);

    this.messageBus.subscribe(
      MessageBusType.TOOL_CONFIRMATION_RESPONSE,
      responseHandler,
    );

    this.messageBus.publish({
      type: MessageBusType.TOOL_CONFIRMATION_REQUEST,
      toolCall,
      correlationId,
      serverName: this._serverName,
    });
  });
}
```

---

## 12. 執行流程圖與安全模型

### 12.1 完整執行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           工具執行完整流程                                   │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. LLM 生成工具調用                                                          │
│    ├── 選擇工具名稱                                                          │
│    ├── 生成參數                                                              │
│    └── 返回 FunctionCall                                                     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. ToolRegistry.getTool(name)                                                │
│    ├── 查找已註冊工具                                                        │
│    ├── 檢查工具是否活躍                                                      │
│    └── 返回 AnyDeclarativeTool 或 undefined                                  │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                    工具存在?     │     工具不存在?
                  ┌──────────────┴──────────────┐
                  │                             │
                  ▼                             ▼
┌──────────────────────────────┐  ┌──────────────────────────────┐
│ 3. Tool.build(params)        │  │ 返回錯誤                      │
│    ├── Schema 驗證           │  │ type: TOOL_NOT_REGISTERED    │
│    ├── 參數值驗證            │  └──────────────────────────────┘
│    └── 創建 ToolInvocation   │
└──────────────┬───────────────┘
               │
          驗證通過?
     ┌─────────┴─────────┐
     │                   │
     ▼                   ▼
┌──────────────┐  ┌──────────────────────────────┐
│ 繼續         │  │ 返回錯誤                      │
│              │  │ type: INVALID_TOOL_PARAMS    │
└──────┬───────┘  └──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. invocation.shouldConfirmExecute(signal)                                   │
│    ├── 查詢 MessageBus 獲取政策決策                                          │
│    │   ├── ALLOW → 跳過確認                                                  │
│    │   ├── DENY → 拋出錯誤                                                   │
│    │   └── ASK_USER → 獲取確認詳情                                           │
│    └── 返回 ToolCallConfirmationDetails 或 false                             │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                    需要確認?     │     不需要確認?
                  ┌──────────────┴──────────────┐
                  │                             │
                  ▼                             │
┌──────────────────────────────┐               │
│ 5. 使用者確認流程             │               │
│    ├── 顯示確認 UI           │               │
│    ├── 等待使用者響應        │               │
│    │   ├── ProceedOnce       │               │
│    │   ├── ProceedAlways     │               │
│    │   ├── ProceedAlwaysAndSave              │
│    │   ├── ModifyWithEditor  │               │
│    │   └── Cancel            │               │
│    └── 更新政策 (如適用)     │               │
└──────────────┬───────────────┘               │
               │                               │
          已批准?                              │
     ┌─────────┴─────────┐                     │
     │                   │                     │
     ▼                   ▼                     │
┌──────────────┐  ┌──────────────┐             │
│ 繼續         │  │ 返回取消     │             │
│              │  │ 或拒絕錯誤   │             │
└──────┬───────┘  └──────────────┘             │
       │                                       │
       └───────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. invocation.execute(signal, updateOutput, config)                          │
│    ├── 執行工具特定邏輯                                                      │
│    │   ├── 檔案系統操作                                                      │
│    │   ├── 程序生成                                                          │
│    │   ├── 網路請求                                                          │
│    │   └── MCP 調用                                                          │
│    ├── 處理中斷信號                                                          │
│    ├── 串流輸出更新                                                          │
│    └── 記錄遙測資料                                                          │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 7. 返回 ToolResult                                                           │
│    ├── llmContent: 給 LLM 的內容                                             │
│    ├── returnDisplay: 給使用者的顯示                                         │
│    └── error?: 錯誤資訊 (如果有)                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 12.2 安全模型

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           安全模型架構                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         防禦層 1: 輸入驗證                            │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │   │
│  │  │ JSON Schema    │  │ 類型強制       │  │ 範圍限制               │  │   │
│  │  │ 驗證參數結構   │  │ 確保類型正確   │  │ 數值/長度限制          │  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         防禦層 2: 路徑安全                            │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │   │
│  │  │ 工作區邊界     │  │ 路徑遍歷防護   │  │ 符號鏈接檢查           │  │   │
│  │  │ 限制訪問範圍   │  │ 阻止 ../ 攻擊  │  │ follow: false          │  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘  │   │
│  │  ┌────────────────┐  ┌────────────────┐                              │   │
│  │  │ 忽略模式       │  │ 臨時目錄       │                              │   │
│  │  │ .gitignore 等  │  │ 允許訪問       │                              │   │
│  │  └────────────────┘  └────────────────┘                              │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         防禦層 3: 命令安全                            │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │   │
│  │  │ 命令白名單     │  │ Shell 包裝剝離 │  │ 根命令提取             │  │   │
│  │  │ isCommandAllowed│ │ stripShellWrapper││ getCommandRoots        │  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘  │   │
│  │  ┌────────────────┐  ┌────────────────┐                              │   │
│  │  │ 避免 eval()    │  │ 環境變數清理   │                              │   │
│  │  │ 禁止動態執行   │  │ sanitizeEnv    │                              │   │
│  │  └────────────────┘  └────────────────┘                              │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         防禦層 4: 網路安全                            │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │   │
│  │  │ 私有 IP 檢測   │  │ 協議白名單     │  │ 超時保護               │  │   │
│  │  │ isPrivateIp    │  │ http/https only│  │ fetchWithTimeout       │  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘  │   │
│  │  ┌────────────────┐                                                  │   │
│  │  │ URL 驗證       │                                                  │   │
│  │  │ parsePrompt    │                                                  │   │
│  │  └────────────────┘                                                  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         防禦層 5: 權限控制                            │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │   │
│  │  │ 政策引擎       │  │ 使用者確認     │  │ 審計日誌               │  │   │
│  │  │ MessageBus     │  │ Confirmation   │  │ Telemetry              │  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘  │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │   │
│  │  │ 批准模式       │  │ MCP 信任級別   │  │ 工具排除               │  │   │
│  │  │ ApprovalMode   │  │ trust flag     │  │ excludeTools           │  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 總結

### 工具系統設計亮點

| 特性 | 描述 |
|------|------|
| **模組化架構** | 23+ 獨立工具，共用 DeclarativeTool 介面 |
| **三層驗證** | Schema → 業務邏輯 → 政策批准 |
| **智慧編輯** | 精確/彈性/正規三策略 + LLM 自我校正 |
| **多傳輸支援** | Stdio、SSE、StreamableHTTP 三種 MCP 傳輸 |
| **豐富遙測** | 檔案操作、編輯策略、錯誤追蹤 |
| **安全優先** | 多層防禦、工作區隔離、命令白名單 |
| **使用者控制** | 完整確認/修改工作流程 |
| **錯誤恢復** | 詳細錯誤類型 + LLM 輔助校正 |

### 工具分類概覽

| 類別 | 工具 | Kind |
|------|------|------|
| 檔案讀取 | ReadFile, ReadManyFiles | Read |
| 檔案編輯 | WriteFile, Edit, SmartEdit | Edit |
| 檔案搜尋 | Glob, Grep, RipGrep | Search |
| 命令執行 | Shell | Execute |
| 網路操作 | WebFetch, WebSearch | Fetch/Search |
| 記憶管理 | Memory | Think |
| 擴展工具 | MCP Tools, Discovered Tools | Other |

本報告全面分析了 Gemini CLI 工具系統的架構設計、實現細節和安全機制，為開發者理解和擴展該系統提供了深入的技術參考。
