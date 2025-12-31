# UI 元件深度分析報告

## 1. 元件層次結構與組織

### 目錄結構概覽

```
ui/
├── App.tsx                          # 入口點 (39 行)
├── AppContainer.tsx                 # 主要狀態容器 (1714 行)
├── components/                      # 120+ UI 元件
│   ├── InputPrompt.tsx             # 輸入欄位 (1235 行)
│   ├── Composer.tsx                # 組合區域
│   ├── DialogManager.tsx           # 對話框管理器
│   ├── MainContent.tsx             # 內容顯示
│   ├── HistoryItemDisplay.tsx      # 訊息渲染器
│   ├── messages/                   # 訊息類型元件
│   │   ├── GeminiMessage.tsx
│   │   ├── UserMessage.tsx
│   │   ├── ToolGroupMessage.tsx
│   │   └── 15+ 其他訊息類型
│   └── shared/                     # 可重用元件
├── contexts/                       # React Context 提供者
├── hooks/                          # 50+ 自訂 hooks
├── layouts/                        # 佈局變體
├── themes/                         # 主題定義
└── utils/                          # 工具與輔助函數
```

---

## 2. 關鍵元件詳細參考

### A. AppContainer (主要狀態容器)

**檔案:** `/packages/cli/src/ui/AppContainer.tsx` (1714 行)

**用途:** 協調所有 UI 狀態和動作的中央狀態管理樞紐

**主要職責 (Lines 163-1235):**
- 初始化和管理所有 context 提供者
- 透過 `useGeminiStream` hook 處理使用者輸入和命令
- 管理對話框狀態 (auth, settings, themes 等)
- 協調鍵盤和滑鼠事件處理
- 管理應用程式生命週期

**狀態管理 (Lines 163-245):**
```typescript
const [quittingMessages, setQuittingMessages] = useState<HistoryItem[] | null>(null);
const [showPrivacyNotice, setShowPrivacyNotice] = useState<boolean>(false);
const [themeError, setThemeError] = useState<string | null>(...);
const [isProcessing, setIsProcessing] = useState<boolean>(false);
const [shellModeActive, setShellModeActive] = useState(false);
const [embeddedShellFocused, setEmbeddedShellFocused] = useState(false);
const [customDialog, setCustomDialog] = useState<React.ReactNode | null>(null);
```

**Provider 嵌套 (封裝):**
- UIStateContext
- UIActionsContext
- KeypressContext
- MouseContext
- ScrollProvider
- VimModeProvider
- SessionStatsProvider

### B. Composer 元件 (輸入區域)

**檔案:** `/packages/cli/src/ui/components/Composer.tsx` (190 行)

**結構 (Lines 33-190):**
```
Composer (Box)
├── LoadingIndicator (line 59)
│   └── 顯示當前思考短語
├── ConfigInitDisplay (line 75)
│   └── 配置狀態
├── QueuedMessageDisplay (line 79)
│   └── 訊息佇列通知
├── TodoTray (line 81)
│   └── 待辦事項
├── ContextSummary Box (lines 83-135)
│   ├── 系統指示器 (line 96)
│   ├── 狀態/警告/上下文摘要 (lines 98-125)
│   ├── AutoAcceptIndicator (line 130)
│   └── ShellModeIndicator (line 132)
└── InputPrompt (line 154)
    └── 主要輸入欄位與建議
```

### C. DialogManager (對話框協調器)

**檔案:** `/packages/cli/src/ui/components/DialogManager.tsx` (237 行)

**對話框渲染優先順序 (Lines 53-234):**
1. IdeTrustChangeDialog - IDE 重啟提示
2. ProQuotaDialog - 配額升級對話框
3. IdeIntegrationNudge - IDE 整合提示
4. FolderTrustDialog - 資料夾信任確認
5. ShellConfirmationDialog - Shell 命令確認
6. LoopDetectionConfirmation - 迴圈偵測對話框
7. ConsentPrompt - 通用確認
8. ThemeDialog - 主題選擇
9. SettingsDialog - 設定編輯器
10. ModelDialog - 模型選擇
11. AuthDialog - 認證流程
12. PrivacyNotice - 隱私披露
13. SessionBrowser - 會話歷史

### D. InputPrompt 元件

**檔案:** `/packages/cli/src/ui/components/InputPrompt.tsx` (1235 行)

**功能 (Lines 65-89):**
- 多行文字編輯與剪貼簿支援
- 使用 `parseInputForHighlighting` 的程式碼高亮
- Shell 歷史補全
- 反向搜尋補全
- Vim 快捷鍵支援
- 建議下拉選單
- 滑鼠支援

**使用的 Hooks:**
```typescript
import { useInputHistory } from '../hooks/useInputHistory.js';
import { useShellHistory } from '../hooks/useShellHistory.js';
import { useReverseSearchCompletion } from '../hooks/useReverseSearchCompletion.js';
import { useCommandCompletion } from '../hooks/useCommandCompletion.js';
import { useKeypress } from '../hooks/useKeypress.js';
import { useMouseClick } from '../hooks/useMouseClick.js';
```

### E. MainContent 元件

**檔案:** `/packages/cli/src/ui/components/MainContent.tsx` (154 行)

**雙重渲染策略:**

**標準模式 (Lines 140-153):**
```typescript
<Static key={uiState.historyRemountKey}>
  <AppHeader key="app-header" />
  ...historyItems
</Static>
{pendingItems}  // 動態待處理訊息
```

**備用緩衝區模式 (Lines 122-137):**
```typescript
<ScrollableList
  hasFocus={!uiState.isEditorDialogOpen}
  data={virtualizedData}
  renderItem={renderItem}
  estimatedItemHeight={() => 100}
  initialScrollIndex={SCROLL_TO_ITEM_END}
/>
```

---

## 3. React Contexts (狀態管理)

### A. UIStateContext

**檔案:** `/packages/cli/src/ui/contexts/UIStateContext.tsx` (152 行)

**UIState 介面 (Lines 45-141):**
```typescript
export interface UIState {
  // 歷史和輸入
  history: HistoryItem[];
  historyManager: UseHistoryManagerReturn;
  buffer: TextBuffer;
  inputWidth: number;
  userMessages: string[];

  // 對話框狀態
  isThemeDialogOpen: boolean;
  isAuthDialogOpen: boolean;
  isSettingsDialogOpen: boolean;
  isSessionBrowserOpen: boolean;
  isModelDialogOpen: boolean;

  // 串流狀態
  streamingState: StreamingState;
  pendingGeminiHistoryItems: HistoryItemWithoutId[];
  pendingHistoryItems: HistoryItemWithoutId[];

  // AI 狀態
  thought: ThoughtSummary | null;
  currentModel: string;
  currentLoadingPhrase: string;
  elapsedTime: number;

  // UI 標記
  shellModeActive: boolean;
  renderMarkdown: boolean;
  copyModeEnabled: boolean;

  // 終端機資訊
  terminalWidth: number;
  terminalHeight: number;
  mainAreaWidth: number;

  // 統計
  sessionStats: SessionStatsState;
}
```

### B. UIActionsContext

**檔案:** `/packages/cli/src/ui/contexts/UIActionsContext.tsx` (70 行)

**UIActions 介面 (Lines 17-59):**
```typescript
export interface UIActions {
  // 主題管理
  handleThemeSelect: (themeName: string, scope: LoadableSettingScope) => void;
  closeThemeDialog: () => void;

  // 認證
  handleAuthSelect: (authType: AuthType | undefined, scope: LoadableSettingScope) => void;
  setAuthState: (state: AuthState) => void;
  onAuthError: (error: string | null) => void;

  // 對話框控制
  closeSettingsDialog: () => void;
  closeModelDialog: () => void;
  openPermissionsDialog: (props?: PermissionsDialogProps) => void;

  // 輸入處理
  handleFinalSubmit: (value: string) => void;
  handleClearScreen: () => void;
  vimHandleInput: (key: Key) => boolean;

  // Shell 模式
  setShellModeActive: (value: boolean) => void;

  // 會話管理
  openSessionBrowser: () => void;
  closeSessionBrowser: () => void;
  handleResumeSession: (session: SessionInfo) => Promise<void>;
}
```

### C. KeypressContext

**檔案:** `/packages/cli/src/ui/contexts/KeypressContext.tsx` (450+ 行)

**按鍵類型定義 (Lines 27-240):**
```typescript
const KEY_INFO_MAP: Record<string, { name: string; shift?: boolean; ctrl?: boolean }> = {
  '[200~': { name: 'paste-start' },
  '[201~': { name: 'paste-end' },
  '[A': { name: 'up' },
  '[B': { name: 'down' },
  '[C': { name: 'right' },
  '[D': { name: 'left' },
  '[3~': { name: 'delete' },
  '[5~': { name: 'pageup' },
  '[6~': { name: 'pagedown' },
  // ... + 60 更多按鍵映射
}
```

### D. ScrollProvider

**檔案:** `/packages/cli/src/ui/contexts/ScrollProvider.tsx` (310+ 行)

**ScrollableEntry 介面 (Lines 26-34):**
```typescript
export interface ScrollableEntry {
  id: string;
  ref: React.RefObject<DOMElement>;
  getScrollState: () => ScrollState;
  scrollBy: (delta: number) => void;
  scrollTo?: (scrollTop: number, duration?: number) => void;
  hasFocus: () => boolean;
  flashScrollbar: () => void;
}
```

### E. VimModeContext

**檔案:** `/packages/cli/src/ui/contexts/VimModeContext.tsx` (80 行)

```typescript
interface VimModeContextType {
  vimEnabled: boolean;
  vimMode: VimMode;  // 'NORMAL' | 'INSERT'
  toggleVimEnabled: () => Promise<boolean>;
  setVimMode: (mode: VimMode) => void;
}
```

---

## 4. 自訂 Hooks (50+ 總計)

### A. useKeypress Hook

**檔案:** `/packages/cli/src/ui/hooks/useKeypress.ts` (37 行)

```typescript
export function useKeypress(
  onKeypress: KeypressHandler,
  { isActive }: { isActive: boolean },
) {
  const { subscribe, unsubscribe } = useKeypressContext();

  useEffect(() => {
    if (!isActive) return;
    subscribe(onKeypress);
    return () => unsubscribe(onKeypress);
  }, [isActive, onKeypress, subscribe, unsubscribe]);
}
```

### B. useGeminiStream Hook (關鍵)

**檔案:** `/packages/cli/src/ui/hooks/useGeminiStream.ts` (1338 行)

**用途:** 管理整個 Gemini 互動生命週期

**返回值 (Lines 1325-1337):**
```typescript
return {
  streamingState: StreamingState,
  submitQuery: async (query, options?, prompt_id?) => void,
  initError: string | null,
  pendingHistoryItems: HistoryItemWithoutId[],
  thought: ThoughtSummary | null,
  cancelOngoingRequest: () => void,
  pendingToolCalls: TrackedToolCall[],
  handleApprovalModeChange: (newApprovalMode: ApprovalMode) => Promise<void>,
  activePtyId: number | undefined,
  loopDetectionConfirmationRequest: {...} | null,
  lastOutputTime: number,
}
```

**串流事件類型 (Lines 805-864):**
```typescript
case ServerGeminiEventType.Thought:
case ServerGeminiEventType.Content:
case ServerGeminiEventType.ToolCallRequest:
case ServerGeminiEventType.UserCancelled:
case ServerGeminiEventType.Error:
case ServerGeminiEventType.ChatCompressed:
case ServerGeminiEventType.Finished:
case ServerGeminiEventType.Citation:
case ServerGeminiEventType.ModelInfo:
case ServerGeminiEventType.LoopDetected:
```

### C. 其他重要 Hooks

| Hook | 用途 |
|------|------|
| `useAlternateBuffer` | 偵測終端機備用螢幕模式 |
| `useTerminalSize` | 追蹤終端機尺寸 |
| `useMouse` | 訂閱滑鼠事件 |
| `useVimMode` | Vim 快捷鍵支援 |
| `useFocus` | 輸入焦點管理 |
| `useShellHistory` | Shell 命令歷史 |
| `useBracketedPaste` | 處理括號貼上事件 |
| `useLoadingIndicator` | 載入短語動畫 |
| `useMessageQueue` | 訊息佇列狀態 |
| `useSessionStats` | 會話指標追蹤 |

---

## 5. Ink 終端機渲染

### A. 佈局系統

**使用的 Ink 元件:**
```typescript
import { Box, Text, Static } from 'ink';
import { measureElement, DOMElement } from 'ink';
import { useStdin, useStdout } from 'ink';
import { useApp } from 'ink';
```

**DefaultAppLayout (Lines 19-66):**
```
DefaultAppLayout
├── Box (root, flexDirection="column", height=terminalHeight-1)
│   ├── MainContent
│   │   ├── AppHeader (static)
│   │   ├── HistoryItemDisplay[] (static)
│   │   └── PendingItems (dynamic)
│   └── MainControls
│       ├── Notifications
│       ├── CopyModeWarning
│       ├── DialogManager 或 Composer
│       └── ExitWarning
```

### B. 虛擬化與滾動

**ScrollableList 元件 (Lines 122-137 in MainContent.tsx):**
```typescript
<ScrollableList
  data={virtualizedData}
  renderItem={renderItem}
  estimatedItemHeight={() => 100}
  keyExtractor={(item, _index) => {...}}
  initialScrollIndex={SCROLL_TO_ITEM_END}
/>
```

### C. MarkdownDisplay 元件

**檔案:** `/packages/cli/src/ui/utils/MarkdownDisplay.tsx` (200+ 行)

**Markdown 功能 (Lines 62-79):**
```typescript
const headerRegex = /^ *(#{1,4}) +(.*)/;
const codeFenceRegex = /^ *(`{3,}|~{3,}) *(\w*?) *$/;
const ulItemRegex = /^([ \t]*)([-*+]) +(.*)/;
const olItemRegex = /^([ \t]*)(\d+)\. +(.*)/;
const tableRowRegex = /^\s*\|(.+)\|\s*$/;
```

---

## 6. 主題系統

### A. 主題結構

**檔案:** `/packages/cli/src/ui/themes/theme.ts` (400+ 行)

**ColorsTheme 介面 (Lines 17-34):**
```typescript
export interface ColorsTheme {
  type: ThemeType;  // 'light' | 'dark' | 'ansi' | 'custom'
  Background: string;
  Foreground: string;
  LightBlue: string;
  AccentBlue: string;
  AccentPurple: string;
  AccentCyan: string;
  AccentGreen: string;
  AccentYellow: string;
  AccentRed: string;
  DiffAdded: string;
  DiffRemoved: string;
  Comment: string;
  Gray: string;
  DarkGray: string;
  GradientColors?: string[];
}
```

### B. 可用主題

- `default.ts` - 深色主題
- `default-light.ts` - 淺色主題
- `ansi.ts` - ANSI 顏色
- `atom-one-dark.ts` - Atom 編輯器顏色
- `ayu.ts` / `ayu-light.ts` - Ayu 主題
- `dracula.ts` - Dracula 主題
- `github-dark.ts` / `github-light.ts` - GitHub 主題
- `holiday.ts` - 節日主題
- `no-color.ts` - 最小主題
- `shades-of-purple.ts` - 紫色主題
- `xcode.ts` - Xcode 主題

### C. 語義顏色

**使用方式:**
```typescript
import { theme } from '../semantic-colors.js';

<Text color={theme.status.error}>錯誤訊息</Text>
<Text color={theme.text.accent}>強調文字</Text>
<Text color={theme.text.response}>AI 回應</Text>
```

---

## 7. 資料流圖

```
使用者輸入 (鍵盤/滑鼠)
    │
    ├─► KeypressContext / MouseContext
    │
    ├─► InputPrompt (文字編輯)
    │
    ├─► useKeypress (訂閱者)
    │
    └─► AppContainer (handleFinalSubmit)
            │
            ├─► useGeminiStream.submitQuery()
            │
            ├─► GeminiClient.sendMessageStream()
            │
            ├─► processGeminiStreamEvents()
            │   ├─ Content 事件 → history
            │   ├─ Tool 請求 → scheduler
            │   ├─ Error 事件 → 錯誤顯示
            │   └─ Citations → 資訊顯示
            │
            └─► historyManager.addItem()
                    │
                    └─► UIState.history (更新)
                        │
                        ├─► MainContent 重新渲染
                        │   └─► HistoryItemDisplay
                        │       └─► 訊息類型元件
                        │           ├─ GeminiMessage
                        │           │  └─ MarkdownDisplay
                        │           ├─ ToolGroupMessage
                        │           └─ ...
                        │
                        └─► Composer 更新
                            └─ InputPrompt 重新啟用
```

---

## 8. 效能優化

### 記憶化策略

1. **元件記憶化 (MainContent.tsx, Line 20):**
   ```typescript
   const MemoizedHistoryItemDisplay = memo(HistoryItemDisplay);
   const MemoizedAppHeader = memo(AppHeader);
   ```

2. **靜態渲染 (MainContent.tsx, Line 142):**
   ```typescript
   <Static key={uiState.historyRemountKey}>
     {/* 渲染一次，永不重新渲染 */}
   </Static>
   ```

3. **昂貴計算的 useMemo (Line 84-91):**
   ```typescript
   const virtualizedData = useMemo(
     () => [
       { type: 'header' as const },
       ...uiState.history.map((item) => ({ type: 'history' as const, item })),
       { type: 'pending' as const },
     ],
     [uiState.history],
   );
   ```

4. **訊息分割 (useGeminiStream.ts, Lines 553-584):**
   - 大訊息在安全點分割
   - 更好的渲染效能
   - 防止閃爍

---

## 9. 關鍵檔案與行數

| 檔案 | 行數 | 用途 |
|------|------|------|
| AppContainer.tsx | 1714 | 主要狀態容器 |
| InputPrompt.tsx | 1235 | 帶建議的輸入欄位 |
| useGeminiStream.ts | 1338 | 串流管理 |
| SettingsDialog.tsx | ~1800 | 設定 UI |
| SessionBrowser.tsx | ~900 | 會話歷史 |
| KeypressContext.tsx | ~450 | 按鍵處理 |
| theme.ts | ~400 | 主題定義 |
| ScrollProvider.tsx | ~310 | 滾動管理 |
| MarkdownDisplay.tsx | ~200 | Markdown 渲染 |

---

## 10. 關鍵架構模式

### 模式 1: Context + Hook 模式
```typescript
// 創建 context
const UIStateContext = createContext<UIState | null>(null);

// 創建消費 hook
export const useUIState = () => {
  const context = useContext(UIStateContext);
  if (!context) throw new Error('...');
  return context;
};

// Provider 包裹樹
<UIStateContext.Provider value={uiState}>
  {children}
</UIStateContext.Provider>
```

### 模式 2: 回調訂閱模式
```typescript
// KeypressContext 使用發布-訂閱
const subscribe = (handler: KeypressHandler) => {
  subscribers.add(handler);
};

const unsubscribe = (handler: KeypressHandler) => {
  subscribers.delete(handler);
};

// useKeypress 在掛載/卸載時訂閱/取消訂閱
useEffect(() => {
  if (!isActive) return;
  subscribe(onKeypress);
  return () => unsubscribe(onKeypress);
}, [isActive, onKeypress, subscribe, unsubscribe]);
```

### 模式 3: 基於 Ref 的效能追蹤
```typescript
const rootUiRef = useRef<DOMElement | null>(null);
const mainControlsRef = useRef<DOMElement | null>(null);

// 用於渲染分析
useFlickerDetector(rootUiRef, terminalHeight);
```
