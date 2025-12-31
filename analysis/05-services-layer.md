# 服務層深度分析報告

## 概覽

`/packages/core/src/services/` 目錄包含 13 個核心服務，處理 Gemini CLI 應用程式的關鍵功能。

---

## 1. Shell 執行服務

**檔案路徑:** `/packages/core/src/services/shellExecutionService.ts`

### 結構與導出

```typescript
export interface ShellExecutionResult {
  rawOutput: Buffer;
  output: string;
  exitCode: number | null;
  signal: number | null;
  error: Error | null;
  aborted: boolean;
  pid: number | undefined;
  executionMethod: 'lydell-node-pty' | 'node-pty' | 'child_process' | 'none';
}

export interface ShellExecutionHandle {
  pid: number | undefined;
  result: Promise<ShellExecutionResult>;
}
```

### 公開 API

**主要執行方法:**
- `static async execute()` - 使用 PTY 或回退執行 shell 命令
  - 透過 callback 支援串流即時輸出

**PTY 管理:**
- `static writeToPty(pid, input)` - 寫入資料到活動 PTY
- `static resizePty(pid, cols, rows)` - 調整終端機尺寸
- `static scrollPty(pid, lines)` - 滾動終端機緩衝區
- `static isPtyActive(pid)` - 檢查 PTY 是否仍在執行

### 內部實現細節

**執行策略:**
1. 首先嘗試透過 `node-pty` 進行 PTY 執行
2. 如果 PTY 不可用則回退到 `child_process`
3. 返回所有輸出作為 Buffers 和解碼字串

**二進位偵測:**
- 嗅探輸出的前 4096 位元組檢查二進位內容
- 如果偵測到二進位則禁用文字解碼和串流
- 為二進位串流發出 `binary_progress` 事件

**輸出緩衝:**
- 每個串流維護 16MB 緩衝區限制 (MAX_CHILD_PROCESS_BUFFER_SIZE)
- 使用 TextDecoder 進行正確的字元集偵測
- 超過限制時截斷並顯示警告訊息

---

## 2. 檔案發現服務

**檔案路徑:** `/packages/core/src/services/fileDiscoveryService.ts`

### 結構與導出

```typescript
export interface FilterFilesOptions {
  respectGitIgnore?: boolean;
  respectGeminiIgnore?: boolean;
}

export interface FilterReport {
  filteredPaths: string[];
  ignoredCount: number;
}
```

### 公開 API

- `filterFiles(paths, options)` - 返回過濾後的列表
- `filterFilesWithReport(paths, options)` - 返回過濾後的列表 + 計數
- `shouldIgnoreFile(path, options)` - 單一檔案的布林檢查

### 內部實現細節

**初始化:**
- 使用 GitIgnoreParser 解析 `.gitignore` (如果是 git 儲存庫)
- 使用 GeminiIgnoreParser 解析 `.geminiignore`
- 創建合併兩者忽略規則的組合過濾器

---

## 3. Git 服務

**檔案路徑:** `/packages/core/src/services/gitService.ts`

### 結構與導出

```typescript
export class GitService {
  async initialize(): Promise<void>;
  async verifyGitAvailability(): Promise<boolean>;
  async setupShadowGitRepository(): Promise<void>;
  async getCurrentCommitHash(): Promise<string>;
  async createFileSnapshot(message: string): Promise<string>;
  async restoreProjectFromSnapshot(commitHash: string): Promise<void>;
}
```

### 內部實現細節

**影子儲存庫:**
- 隱藏的 git 儲存庫在 `~/.gemini/tmp/<project_hash>/`
- 專用 `.gitconfig` 防止使用者偏好影響檢查點
- 用於透明的專案快照

**環境隔離:**
```javascript
GIT_DIR: <history_dir>/.git
GIT_WORK_TREE: <project_root>
HOME: <history_dir>  // 防止使用者配置繼承
XDG_CONFIG_HOME: <history_dir>
```

---

## 4. 聊天記錄服務

**檔案路徑:** `/packages/core/src/services/chatRecordingService.ts`

### 結構與導出

```typescript
export interface ConversationRecord {
  sessionId: string;
  projectHash: string;
  startTime: string;
  lastUpdated: string;
  messages: MessageRecord[];
  summary?: string;
}

export interface TokensSummary {
  input: number;
  output: number;
  cached: number;
  thoughts?: number;
  tool?: number;
  total: number;
}
```

### 公開 API

**初始化:**
- `initialize(resumedSessionData?)` - 創建新會話或恢復現有會話

**記錄方法:**
- `recordMessage(message)` - 記錄使用者/助手/系統訊息
- `recordThought(thought)` - 記錄助手思考
- `recordMessageTokens(metadata)` - 更新 token 計數
- `recordToolCalls(model, calls)` - 記錄工具執行

**檔案結構:**
- 位置: `~/.gemini/tmp/<project_hash>/chats/session-*.json`
- 檔名格式: `session-YYYY-MM-DDTHH-MM-<uuid>.json`

---

## 5. 會話摘要服務

**檔案路徑:** `/packages/core/src/services/sessionSummaryService.ts`

### 公開 API

```typescript
async generateSummary(options: GenerateSummaryOptions): Promise<string | null>
```

### 內部實現細節

**演算法:**
1. 僅過濾 user/gemini 訊息
2. 應用滑動視窗選擇: `ceil(max/2) 前 + floor(max/2) 後`
3. 截斷長訊息到 500 字元
4. 格式化為 "Role: content" 對
5. 使用特定提示呼叫 LLM

**配置:**
```typescript
const DEFAULT_MAX_MESSAGES = 20;        // 包含的訊息數
const DEFAULT_TIMEOUT_MS = 5000;        // 請求超時
const MAX_MESSAGE_LENGTH = 500;         // 每訊息截斷
```

---

## 6. 模型配置服務

**檔案路徑:** `/packages/core/src/services/modelConfigService.ts`

### 結構與導出

```typescript
export interface ModelConfigKey {
  model: string;
  overrideScope?: string;
  isRetry?: boolean;
}

export interface ModelConfig {
  model?: string;
  generateContentConfig?: GenerateContentConfig;
}
```

### 公開 API

- `registerRuntimeModelConfig(name, alias)` - 添加執行時別名
- `getResolvedConfig(context)` - 返回已解析的配置 + 模型名稱

### 解析管道

1. **別名解析** - 遞迴解析別名鏈，偵測循環依賴
2. **覆蓋應用** - 匹配覆蓋規則，按特定性排序
3. **深度合併** - 遞迴合併物件，陣列被替換

---

## 7. 迴圈偵測服務

**檔案路徑:** `/packages/core/src/services/loopDetectionService.ts`

### 公開 API

```typescript
export class LoopDetectionService {
  addAndCheck(event: ServerGeminiStreamEvent): boolean;
  async turnStarted(signal: AbortSignal): Promise<boolean>;
  disableForSession(): void;
  reset(promptId: string): void;
}
```

### 偵測方法

**1. 工具呼叫迴圈 (閾值: 5)**
- 偵測重複的相同工具呼叫
- 使用呼叫名稱 + 參數的 SHA256 雜湊
- 連續 5+ 相同呼叫時觸發

**2. 內容迴圈 (閾值: 10)**
- 分析串流文字的重複
- 以 50 字元視窗分塊
- 當 chunk 出現 10+ 次時偵測迴圈
- 檢查聚類: 平均距離 ≤ 250 字元

**3. 基於 LLM 的迴圈檢查**
- 在提示 30 turns 後觸發
- 週期性執行 (預設: 每 3 turns)
- 使用 Flash 模型進行初始檢查

---

## 8. 技能管理器

**檔案路徑:** `/packages/core/src/services/skillManager.ts`

### 公開 API

```typescript
export class SkillManager {
  async discoverSkills(storage: Storage): Promise<void>;
  getSkills(): SkillMetadata[];
  getAllSkills(): SkillMetadata[];
  getSkill(name: string): SkillMetadata | null;
  setDisabledSkills(disabledNames: string[]): void;
  activateSkill(name: string): void;
}
```

### 技能檔案格式

```markdown
---
name: SkillName
description: 技能描述
---
技能內容 (Markdown)
```

### 發現程序
1. 在使用者技能目錄搜尋 `*/SKILL.md`
2. 在專案技能目錄搜尋 `*/SKILL.md`
3. 專案技能覆蓋使用者技能 (按名稱)
4. 解析 YAML frontmatter + markdown 內容

---

## 9. 上下文管理器

**檔案路徑:** `/packages/core/src/services/contextManager.ts`

### 公開 API

```typescript
export class ContextManager {
  async refresh(): Promise<void>;
  async discoverContext(accessedPath: string, trustedRoots: string[]): Promise<string>;
  getGlobalMemory(): string;
  getEnvironmentMemory(): string;
}
```

### 三層記憶系統

1. **第 1 層: 全域記憶** - 啟動時載入，適用於所有操作
2. **第 2 層: 環境記憶** - 啟動時載入，每個工作區上下文
3. **第 3 層: JIT 子目錄記憶** - 存取檔案時按需載入

---

## 10. 聊天壓縮服務

**檔案路徑:** `/packages/core/src/services/chatCompressionService.ts`

### 結構與導出

```typescript
export const DEFAULT_COMPRESSION_TOKEN_THRESHOLD = 0.5;
export const COMPRESSION_PRESERVE_THRESHOLD = 0.3;

export function findCompressSplitPoint(
  contents: Content[],
  fraction: number,
): number;
```

### 壓縮流程

1. **檢查條件** - 空歷史、失敗嘗試、閾值
2. **找到分割點** - 計算字元數，保留最後 30%
3. **壓縮歷史** - 發送舊歷史到 LLM 生成摘要
4. **計算 Tokens** - 驗證壓縮是否有效

---

## 11. 環境淨化服務

**檔案路徑:** `/packages/core/src/services/environmentSanitization.ts`

### 公開 API

```typescript
export function sanitizeEnvironment(
  processEnv: NodeJS.ProcessEnv,
  config: EnvironmentSanitizationConfig,
): NodeJS.ProcessEnv;
```

### 始終允許的變數 (18 個)
```
PATH, SYSTEMROOT, COMSPEC, PATHEXT, WINDIR, TEMP, TMP
USERPROFILE, SYSTEMDRIVE, HOME, LANG, SHELL, TMPDIR
USER, LOGNAME, GITHUB_ENV 等
```

### 從不允許的變數 (12+ 個)
```
CLIENT_ID, DB_URI, CONNECTION_STRING, AWS_DEFAULT_REGION
AZURE_CLIENT_ID, SLACK_WEBHOOK_URL, DATABASE_URL 等
```

### 名稱模式 (從不允許)
- TOKEN, SECRET, PASSWORD, PASSWD
- KEY, AUTH, CREDENTIAL, CREDS
- PRIVATE, CERT

---

## 服務整合圖

```
┌─────────────────────────────────────────────────────────────┐
│              應用程式核心                                    │
├─────────────────────────────────────────────────────────────┤
│  Config ←→ BaseLlmClient ←→ GeminiChat                      │
└──────────┬──────────────────────────────────────────────────┘
           │
     ┌─────┴─────────────────────────────────────────────────┐
     │              服務層                                    │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 聊天記錄 - 對話持久化到磁碟                      │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 會話摘要 - 透過 LLM 生成 1 行摘要               │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ Shell 執行 - 透過 PTY 或 child_process 執行     │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 迴圈偵測 - 防止無限迴圈                          │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 聊天壓縮 - Token 感知壓縮                        │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 模型配置 - 別名解析與覆蓋應用                    │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ Git 服務 - 影子儲存庫與檢查點                    │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 檔案發現 - .gitignore/.geminiignore 解析         │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 技能管理器 - SKILL.md 檔案發現                   │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 上下文管理器 - 三層記憶管理                      │  │
     │  └─────────────────────────────────────────────────┘  │
     │  ┌─────────────────────────────────────────────────┐  │
     │  │ 環境淨化 - 憑證過濾                              │  │
     │  └─────────────────────────────────────────────────┘  │
     └─────────────────────────────────────────────────────────┘
```

---

## 跨服務模式

### 1. 錯誤處理策略
- **Shell 執行**: 回退鏈 (PTY → child_process)
- **聊天記錄**: 優雅降級，記錄錯誤
- **會話摘要**: 失敗時返回 null
- **Git**: 關鍵失敗時拋出描述性錯誤
- **模型配置**: 僅在無法解析時拋出

### 2. 非同步模式
- 全程基於 Promise 的 API
- 支援 AbortSignal 取消
- 使用 AbortController 進行超時管理
- 有序操作的處理鏈

### 3. 配置管理
- 集中的 Config 服務依賴
- 覆蓋/別名層次結構
- 整合使用者偏好
- 模型感知配置

### 4. 持久化
- JSON 檔案儲存 (聊天、會話)
- Git 基礎快照 (專案狀態)
- 適當的記憶體快取
- 競態條件處理 (重新讀取後寫入)

### 5. 安全性
- 環境變數淨化
- 憑證模式偵測
- GitHub Actions 嚴格模式
- 程序隔離 (Git 影子儲存庫)
