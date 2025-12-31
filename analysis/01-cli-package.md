# CLI Package 深度分析報告

## 1. 入口點與初始化流程

### 主要入口點鏈
```
/home/user/gemini-cli/packages/cli/index.ts (line 9)
  ↓ 呼叫 main()
/home/user/gemini-cli/packages/cli/src/gemini.tsx (line 293-709)
  ↓ 主要協調函數
```

### 入口流程圖
```
index.ts (#!/usr/bin/env node)
    ↓
    main() in gemini.tsx
    ├─ 設置未處理的 rejection handler (line 157-177)
    ├─ 載入設定 (line 303-310)
    ├─ 解析 CLI 參數 (line 335-337)
    ├─ 檢查沙盒需求 (line 381-472)
    ├─ 載入完整 CLI 配置 (line 478-482)
    ├─ 初始化應用元件 (line 575-577)
    ├─ 判斷互動 vs 非互動模式 (line 622)
    │   ├─ 如果互動: startInteractiveUI() (line 623-631)
    │   └─ 如果非互動: runNonInteractive() (line 697-704)
    └─ 清理和退出 (line 706-708)
```

### 關鍵函數
- **`main()`** (gemini.tsx:293-709): 主要入口協調器
- **`startInteractiveUI()`** (gemini.tsx:179-291): 渲染 React/Ink UI
- **`runNonInteractive()`** (nonInteractiveCli.ts:58-65): 處理腳本模式

---

## 2. UI 元件層次結構與架構

### 元件樹結構
```
AppContainer.tsx (1714 行)
├─ 狀態管理 (useReducer, useCallback hooks)
├─ Context 設置
│   ├─ AppContext.Provider
│   ├─ UIStateContext.Provider
│   ├─ UIActionsContext.Provider
│   └─ ConfigContext.Provider
├─ App 元件
│   ├─ DefaultAppLayout (標準模式)
│   │   ├─ MainContent
│   │   │   └─ ChatList, HistoryDisplay
│   │   ├─ Notifications
│   │   ├─ DialogManager
│   │   │   ├─ ThemeDialog
│   │   │   ├─ AuthDialog
│   │   │   ├─ ModelDialog
│   │   │   ├─ SettingsDialog
│   │   │   └─ 更多...
│   │   └─ Composer
│   │       ├─ LoadingIndicator
│   │       ├─ InputPrompt
│   │       └─ Footer
│   └─ ScreenReaderAppLayout (無障礙模式)
└─ QuittingDisplay (退出時)
```

### 主要元件 (按大小/重要性)

| 元件 | 行數 | 用途 |
|------|------|------|
| `AppContainer.tsx` | 1714 | 主要狀態容器、協調 |
| `App.tsx` | 38 | 佈局路由 |
| `DefaultAppLayout.tsx` | ~80+ | 主要 UI 佈局組合 |
| `Composer.tsx` | 80+ | 輸入區域與指示器 |
| `MarkdownDisplay.tsx` | 80+ | Markdown 渲染引擎 |

### 訊息元件類型
```
GeminiMessage.tsx        - LLM 回應與 markdown
UserMessage.tsx          - 使用者提示顯示
ToolMessage.tsx          - 工具呼叫輸出
ShellToolMessage.tsx     - Shell 命令執行
ToolConfirmationMessage  - 確認對話框
ErrorMessage.tsx         - 錯誤顯示
WarningMessage.tsx       - 警告通知
```

---

## 3. 命令系統：Slash Commands 架構

### 命令載入管道
```
CommandService.create() [services/CommandService.ts:48-96]
    ↓
    從多個 Loader 並行載入
    ├─ BuiltinCommandLoader (同步)
    │   └─ 30+ 硬編碼命令來自 /ui/commands/
    ├─ FileCommandLoader (非同步，檔案系統掃描)
    │   ├─ 使用者命令: ~/.config/gemini/commands/
    │   ├─ 專案命令: ./.gemini/commands/
    │   └─ 擴展命令: extension_dir/commands/
    └─ McpPromptLoader (非同步)
        └─ MCP 伺服器 prompts
    ↓
    衝突解決
    ↓
    CommandService 實例與合併命令
```

### BuiltinCommandLoader (lines 51-102)
載入 30+ 硬編碼命令：
```
aboutCommand, authCommand, bugCommand, chatCommand, clearCommand,
compressCommand, copyCommand, docsCommand, extensionsCommand,
helpCommand, hooksCommand, mcpCommand, memoryCommand, modelCommand,
settingsCommand, skillsCommand, statsCommand, themeCommand, toolsCommand...
```

### 命令類型

| 類型 | 種類 | 範例 |
|------|------|------|
| Builtin | CODE | /help, /chat, /settings |
| File-based | FILE | 自訂 .toml 命令 |
| Extension | FILE | 從 extension/commands/ |
| MCP Prompt | MCP | 伺服器提供的 prompts |

---

## 4. 配置載入系統

### 設定載入層次
```
/home/user/gemini-cli/packages/cli/src/config/

settings.ts (897 行) - 核心設定載入
    ↓
    loadSettings()
        ├─ 從 ~/.config/gemini/settings.json (USER) 載入
        ├─ 從 .gemini/settings.json (PROJECT) 載入
        ├─ 與環境變數合併
        ├─ 應用棄用鍵的遷移映射
        ├─ 根據 schema 驗證
        └─ 返回 LoadedSettings 物件

settingsSchema.ts (2049 行)
    └─ 完整的 JSON Schema 定義與類型
```

### 設定結構
```typescript
LoadedSettings {
  merged: Settings              // 最終合併配置
  user: Settings                // 使用者級別設定
  workspace: Settings           // 專案級別設定
  errors: ValidationError[]     // 驗證問題
  setValue(scope, path, value)  // 更新並儲存
  getScopedValue(scope, path)   // 從特定級別取得
}
```

### 設定範圍
```
User:      ~/.config/gemini/settings.json
Project:   .gemini/settings.json
Workspace: (記憶體中的 UI 覆蓋)
```

---

## 5. 狀態管理：React Contexts & Hooks

### Context 架構

**主要 Contexts (在 /src/ui/contexts/):**

| Context | 用途 | 關鍵資料 |
|---------|------|---------|
| `UIStateContext` | 主要 UI 狀態 | history, streamingState, buffer, 30+ 屬性 |
| `UIActionsContext` | 動作處理器 | handleThemeSelect, handleFinalSubmit, 20+ 處理器 |
| `SettingsContext` | 使用者設定 | LoadedSettings |
| `ConfigContext` | 執行時配置 | Config 物件 |
| `VimModeContext` | Vim 模式狀態 | vimEnabled, vimMode |
| `KeypressContext` | 鍵盤輸入 | Key 事件廣播 |
| `MouseContext` | 滑鼠事件 | SGR 滑鼠序列解析 |
| `SessionContext` | 會話統計 | token 計數, 工具呼叫統計 |
| `StreamingContext` | 串流狀態 | isStreaming, streamPhase |

### 狀態流程圖
```
使用者輸入 (鍵盤/滑鼠)
    ↓
KeypressContext 廣播 Key 事件
    ↓
useKeypress() hook 在活動元件
    ↓
元件更新本地狀態或呼叫 UIAction
    ↓
UIActions 更新 UIStateContext
    ↓
所有使用 UIState 的元件重新渲染
    ↓
透過 Ink 渲染視覺更新
    ↓
顯示到終端機
```

---

## 6. 輸入處理系統

### 鍵盤與滑鼠輸入管道

**KeypressContext (KeypressContext.tsx:249-625):**

```
stdin.on('data', dataListener)
    ↓
createDataListener()
    ├─ 緩衝 escape 序列
    ├─ 偵測多位元組字元
    └─ emitKeys() 生成器
    ↓
emitKeys() [Line 270-529]
    ├─ 解析 ANSI escape 序列
    ├─ 偵測特殊鍵 (方向鍵, F1-F12 等)
    ├─ 處理修飾鍵 (Ctrl, Shift, Meta)
    └─ 生成 Key 物件
    ↓
廣播給訂閱者
    ↓
使用 useKeypress(handler, {isActive}) 的元件
```

### 特殊鍵處理

| 輸入 | 偵測 | 處理 |
|------|------|------|
| `Ctrl+C` | name:'c', ctrl:true | 退出確認 |
| `Escape` | name:'escape' | 關閉對話框 |
| `Tab` | name:'tab' | 命令補全 |
| `\<Return>` | 反斜線緩衝 | 多行提示 |
| 滑鼠 | SGR 序列 | 滾動/點擊 |

---

## 7. 輸出渲染系統

### Markdown 渲染管道

**MarkdownDisplay.tsx:**
```
文字輸入
    ↓
按行分割
    ↓
解析區塊結構:
├─ 標題 (# ## ### ####)
├─ 程式碼區塊 (``` 或 ~~~)
├─ 表格 (| col | col |)
├─ 列表 (- 或 * 或 1.)
├─ 水平線 (---)
├─ 引用區塊 (>)
└─ 段落
    ↓
使用 CodeColorizer 渲染
├─ highlight.js 語法高亮
├─ 行號 (可選)
└─ 語言偵測
    ↓
透過 Ink <Box> 和 <Text> 元件輸出
```

---

## 8. 非互動模式

### 非互動 CLI 執行流程

**runNonInteractive() (nonInteractiveCli.ts:58-200+):**

```
runNonInteractive({config, settings, input, prompt_id})
    ↓
    設置主控台修補
    ├─ 重定向主控台日誌到 stderr
    ├─ 設置文字輸出寫入器
    └─ 註冊錯誤處理器
    ↓
    初始化 Gemini 串流
    ├─ 創建 GeminiChat 實例
    ├─ 如果恢復則載入記憶
    └─ 初始化工具處理器
    ↓
    處理使用者輸入命令
    ├─ 檢查是否 slash 命令
    ├─ 檢查是否 @ 命令
    └─ 否則: 作為提示發送
    ↓
    串流 Gemini 回應
    ├─ 處理工具呼叫
    ├─ 串流到 stdout
    └─ 追蹤指標
    ↓
    輸出格式化
    ├─ JSON 模式
    ├─ 文字模式
    └─ 結構化格式
    ↓
    退出和清理
```

### 輸出格式
- `text` - 純 markdown (預設)
- `json` - 結構化 JSON 輸出
- `stream-json` - 換行分隔的 JSON
- `raw` - 原始串流文字

---

## 9. IDE 整合

### Zed 編輯器整合 (zed-integration/zedIntegration.ts)

**架構:**
```
ACP 協議 (Agent Client Protocol)
    ↓
GeminiAgent (Zed 的代理介面)
├─ initialize(caps)          - 握手
├─ createSession()           - 新聊天
├─ completeContent()         - 生成回應
├─ callTool()                - 執行工具
└─ listResources()           - 可用資源
```

---

## 10. 元件大小參考

```
大型元件 (500+ 行):
├─ AppContainer.tsx         1714 行
├─ gemini.tsx              709 行
├─ useGeminiStream.ts      600+ 行
├─ DefaultAppLayout.tsx    500+ 行
└─ MarkdownDisplay.tsx     400+ 行

中型元件 (100-500 行):
├─ Composer.tsx             200+ 行
├─ MainContent.tsx          300+ 行
├─ DialogManager.tsx        300+ 行
└─ 40+ 其他

小型元件 (<100 行):
├─ GeminiMessage.tsx        54 行
├─ UserMessage.tsx          40 行
└─ 100+ 其他
```

---

## 資料流架構總結

### 互動模式完整流程

```
使用者動作 (鍵盤輸入)
    ↓
KeypressContext.broadcast()
    ↓
活動元件中的 Keypress 訂閱者
    ↓
    ┌─────────────────┼─────────────────┐
    ↓                 ↓                 ↓
Vim 處理器      命令處理器      文字輸入
    ↓                 ↓                 ↓
    └─────────────────┼─────────────────┘
                      ↓
            UIActions (從 context)
                      ↓
            UIStateContext.update()
                      ↓
            重新渲染元件
                      ↓
            顯示到終端機
```
