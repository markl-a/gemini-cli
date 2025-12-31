# 工具系統深度分析報告

## 執行摘要

工具系統是一個複雜的模組化框架，用於在 Gemini CLI 中執行操作。它包含 **23 個主要工具類別**，具有全面的安全機制、批准工作流程和智慧執行策略。

**關鍵指標:**
- **總實現檔案**: 49 個 TypeScript 檔案 (23 實現, 26 測試)
- **總程式碼行數**: ~15,000+ 行
- **工具類別**: 8 大類別
- **安全機制**: 3 層驗證 (schema、參數、執行)

---

## 1. 工具註冊與發現系統

### 檔案: `/packages/core/src/tools/tool-registry.ts` (533 行)

#### 架構

```
ToolRegistry
├── allKnownTools: Map<string, AnyDeclarativeTool>
├── config: Config
└── messageBus?: MessageBus

DiscoveredTool extends BaseDeclarativeTool
├── originalName: string
├── parameterSchema: Record<string, unknown>
└── DiscoveredToolInvocation
```

#### 工具發現流程 (Lines 293-422)

```
1. discoverAllTools()
   ├── removeDiscoveredTools()
   └── discoverAndRegisterToolsFromCommand()
       ├── 解析發現命令
       ├── 生成子程序
       ├── 串流 stdout/stderr (限制 10MB)
       ├── 解析工具 JSON 陣列
       ├── 提取 FunctionDeclarations
       └── 註冊 DiscoveredTool 實例
```

#### 工具組織 (Lines 239-266)

**排序優先順序:**
1. **內建工具** (優先順序 0)
2. **發現的工具** (優先順序 1)
3. **MCP 工具** (優先順序 2, 按伺服器名稱排序)

---

## 2. 基礎框架與工具介面

### 檔案: `/packages/core/src/tools/tools.ts` (756 行)

#### 核心介面

**ToolInvocation** (Lines 25-66):
```typescript
interface ToolInvocation<TParams, TResult> {
  params: TParams
  getDescription(): string
  toolLocations(): ToolLocation[]
  shouldConfirmExecute(signal): Promise<ConfirmationDetails | false>
  execute(signal, updateOutput?, config?): Promise<TResult>
}
```

#### 工具分類 (Lines 730-756)

**Kind 列舉:**
- `Read` - 檔案讀取操作
- `Edit` - 檔案修改
- `Delete` - 檔案刪除
- `Move` - 檔案移動
- `Search` - 內容/檔案搜尋
- `Execute` - Shell/子程序執行
- `Think` - 分析/推理
- `Fetch` - 遠端資源獲取
- `Other` - 其他

---

## 3. 檔案工具

### 3.1 ReadFile 工具

**檔案**: `/packages/core/src/tools/read-file.ts` (240 行)

#### 參數
```typescript
interface ReadFileToolParams {
  file_path: string      // 必須: 要讀取的檔案
  offset?: number        // 可選: 0-based 行起始位置
  limit?: number         // 可選: 最大讀取行數
}
```

#### 執行流程 (Lines 76-138)
- 處理文字、圖片、音訊、PDF
- 支援分頁讀取
- 記錄遙測資料

### 3.2 WriteFile 工具

**檔案**: `/packages/core/src/tools/write-file.ts` (527 行)

#### 參數
```typescript
interface WriteFileToolParams {
  file_path: string           // 必須
  content: string             // 必須
  modified_by_user?: boolean  // 使用者修改標記
  ai_proposed_content?: string // 原始 AI 提案
}
```

#### 核心函數: getCorrectedFileContent (Lines 81-141)
- 驗證並校正提議的內容
- 使用 LLM 進行校正
- 處理新檔案 vs 現有檔案

### 3.3 Edit (Replace) 工具

**檔案**: `/packages/core/src/tools/edit.ts` (629 行)

#### 參數
```typescript
interface EditToolParams {
  file_path: string              // 必須
  old_string: string             // 必須 (空 = 創建新檔案)
  new_string: string             // 必須
  expected_replacements?: number // 預設為 1
}
```

#### calculateEdit() 流程 (Lines 142-259)
1. 正規化行結尾 (CRLF → LF)
2. 讀取檔案內容
3. 偵測新檔案
4. 透過 LLM 進行編輯校正
5. 驗證檢查
6. 應用替換
7. 最終驗證

---

## 4. 搜尋工具

### 4.1 Glob 工具

**檔案**: `/packages/core/src/tools/glob.ts`

#### 參數
```typescript
interface GlobToolParams {
  pattern: string              // 必須: glob 模式
  dir_path?: string            // 可選: 搜尋目錄
  case_sensitive?: boolean     // 預設 false
  respect_git_ignore?: boolean // 預設 true
}
```

### 4.2 Grep 工具

**檔案**: `/packages/core/src/tools/grep.ts`

#### 參數
```typescript
interface GrepToolParams {
  pattern: string       // 要搜尋的正規表達式模式
  dir_path?: string     // 可選: 搜尋目錄
  include?: string      // 可選: 檔案模式
}
```

### 4.3 RipGrep 工具

**檔案**: `/packages/core/src/tools/ripGrep.ts`

- 使用原生編譯的 ripgrep 二進位檔
- 內建 .gitignore 支援
- 更好的 Unicode 處理
- 平行檔案處理

---

## 5. Shell 工具

**檔案**: `/packages/core/src/tools/shell.ts` (542 行)

#### 參數
```typescript
interface ShellToolParams {
  command: string      // 必須: shell 命令
  description?: string // 可選: 人類可讀描述
  dir_path?: string    // 可選: 工作目錄
}
```

#### 執行策略 (Lines 100+)
1. 命令驗證與白名單檢查
2. 政策更新追蹤
3. 透過 ShellExecutionService 執行
4. 輸出串流
5. 錯誤處理

#### 安全機制
- 權限檢查
- Shell wrapper 剝離
- 根命令提取
- 避免 eval() 使用

---

## 6. Web 工具

### 6.1 WebFetch 工具

**檔案**: `/packages/core/src/tools/web-fetch.ts`

#### 參數
```typescript
interface WebFetchToolParams {
  url: string        // 必須: 要獲取的 URL
  prompt?: string    // 可選: 處理提示
}
```

#### 執行流程
1. URL 驗證
2. 帶超時的獲取 (10 秒)
3. 私有 IP 保護
4. HTML → 文字轉換
5. Grounding 元資料提取

### 6.2 WebSearch 工具

**檔案**: `/packages/core/src/tools/web-search.ts`

- 整合 Google 搜尋 grounding
- 支援引用
- 快取搜尋結果

---

## 7. MCP 工具

### 7.1 MCP 工具包裝器

**檔案**: `/packages/core/src/tools/mcp-tool.ts` (446 行)

```typescript
class DiscoveredMCPToolInvocation extends BaseToolInvocation<
  ToolParams,
  ToolResult
> {
  mcpTool: CallableTool        // 來自 MCP SDK
  serverName: string           // MCP 伺服器名稱
  serverToolName: string       // 伺服器中的工具名稱
  displayName: string          // 使用者友好名稱
  trust?: boolean              // 信任級別標記
}
```

### 7.2 MCP 客戶端

**檔案**: `/packages/core/src/tools/mcp-client.ts` (1,846 行)

#### 支援的傳輸層
- StdioClientTransport
- SSEClientTransport
- StreamableHTTPClientTransport

---

## 8. 記憶工具

**檔案**: `/packages/core/src/tools/memoryTool.ts`

#### 用途
在 GEMINI.md 檔案中持久化長期記憶

#### 使用準則
**使用時機:**
- 使用者明確要求記住某事
- 關於使用者偏好/環境的重要事實
- 自包含、清晰的陳述

**不使用時機:**
- 僅限會話的上下文
- 長/複雜/冗長的文字
- 不確定重要性

---

## 9. 智慧編輯工具

**檔案**: `/packages/core/src/tools/smart-edit.ts` (1,013 行)

#### 策略

**1. 精確替換** (Lines 91-118):
- 正規化行結尾
- 計算精確出現次數
- 應用文字替換

**2. 彈性替換** (Lines 120-180):
- 從搜尋行去除空白
- 滑動視窗匹配
- 重新應用縮進

**3. LLM 基礎修復** (Lines 200+):
- 當精確和彈性都失敗時
- 使用 LLM 校正編輯

---

## 10. 錯誤處理系統

**檔案**: `/packages/core/src/tools/tool-error.ts` (104 行)

### ToolErrorType 列舉

**一般錯誤:**
- `INVALID_TOOL_PARAMS`
- `UNKNOWN`
- `TOOL_NOT_REGISTERED`
- `EXECUTION_FAILED`

**檔案系統錯誤:**
- `FILE_NOT_FOUND`
- `PERMISSION_DENIED`
- `NO_SPACE_LEFT`
- `PATH_NOT_IN_WORKSPACE`

**編輯錯誤:**
- `EDIT_NO_OCCURRENCE_FOUND`
- `EDIT_EXPECTED_OCCURRENCE_MISMATCH`
- `EDIT_NO_CHANGE`

---

## 架構圖

```
┌─────────────────────────────────────────────────────────────┐
│                      Tool Registry                          │
│  管理註冊、發現和工具生命週期                                │
└───────────────────┬─────────────────────────────────────────┘
                    │
        ┌───────────┴────────────┬────────────┬──────────┐
        │                        │            │          │
   ┌────▼─────┐         ┌────────▼───┐   ┌───▼────┐  ┌──▼───┐
   │ Declarative│       │Discovered   │   │Modifi  │  │ MCP  │
   │  Tool      │       │   Tools     │   │ able   │  │ Tool │
   └────┬─────┘        └────────┬──┘    └───┬────┘  └──┬───┘
        │                       │           │          │
        ├───────────────────────┼───────────┼──────────┤
        │                       │           │          │
    執行 & 串流             確認        政策更新    遙測
```

---

## 執行流程圖

```
使用者請求
    │
    ▼
┌─────────────────────────────────────┐
│ LLM 識別要使用的工具                │
│ 生成參數                            │
└────────────────────┬────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ Tool.build(params)         │
        │ ├─ JSON schema 驗證        │
        │ ├─ 參數值檢查              │
        │ └─ 創建 ToolInvocation     │
        └────────────────┬───────────┘
                         │
                         ▼
        ┌────────────────────────────────────┐
        │ invocation.shouldConfirmExecute()   │
        │ ├─ 檢查 MessageBus 政策            │
        │ ├─ 判斷 ALLOW/DENY/ASK_USER        │
        │ └─ 如果 ASK 則獲取確認詳情         │
        └────────────────┬───────────────────┘
                         │
                    ┌────┴────┐
                    │          │
            已批准  │          │未批准
                    │          │
                    ▼          ▼
        ┌────────────┐    ┌──────────┐
        │  執行工具  │    │ 拒絕 &   │
        │            │    │ 返回錯誤 │
        └─────┬──────┘    └──────────┘
              │
              ▼
    ┌──────────────────────────┐
    │ 工具特定執行             │
    │ ├─ 檔案系統操作          │
    │ ├─ 程序生成              │
    │ ├─ 網路請求              │
    │ └─ 輸出串流              │
    └─────────────┬────────────┘
                  │
                  ▼
        ┌──────────────────┐
        │ ToolResult       │
        │ ├─ llmContent    │
        │ ├─ returnDisplay │
        │ └─ error?        │
        └──────────────────┘
```

---

## 安全模型

### 三層驗證

```
第 1 層: Schema 驗證
├── JSON schema 驗證
├── 類型檢查
├── 必填欄位檢查
└── 無效時立即拒絕

第 2 層: 參數驗證
├── 業務邏輯檢查
├── 工作區邊界檢查
├── 路徑遍歷防護
├── 檔案存在檢查
└── 工具特定約束

第 3 層: 執行批准
├── 政策決策 (ALLOW/DENY/ASK_USER)
├── 需要時使用者確認
├── 政策持久化
└── 審計日誌
```

---

## 結論

工具系統是一個**成熟的生產級框架**，具備:

1. **模組化架構**: 23+ 獨立工具，共用介面
2. **全面安全**: 多層驗證 + 批准工作流程
3. **智慧編輯**: 多種策略 (精確、彈性、LLM 驅動)
4. **可擴展性**: MCP 協議支援第三方工具整合
5. **豐富遙測**: 檔案操作、智慧編輯策略、錯誤追蹤
6. **使用者控制**: 完整確認/修改功能
7. **錯誤恢復**: 詳細錯誤類型 + LLM 自我校正支援
8. **串流支援**: 長時間操作的即時輸出更新
