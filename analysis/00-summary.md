# Gemini CLI 專案深度分析總結

## 專案概覽

**名稱:** Gemini CLI
**版本:** 0.24.0-nightly
**授權:** Apache 2.0
**描述:** Google 的開源 AI 代理，將 Gemini 直接帶入終端機

---

## 架構摘要

### 雙層架構

```
┌─────────────────────────────────────────────────────────────┐
│                    CLI 套件 (前端)                           │
│  • React/Ink 終端機 UI                                      │
│  • 命令解析與路由                                            │
│  • 使用者輸入處理                                            │
│  • 主題與視覺渲染                                            │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    Core 套件 (後端)                          │
│  • Gemini API 客戶端                                        │
│  • 工具系統                                                  │
│  • Hooks 引擎                                               │
│  • 代理框架                                                  │
│  • 會話管理                                                  │
└─────────────────────────────────────────────────────────────┘
```

### 技術棧

| 類別 | 技術 |
|------|------|
| 語言 | TypeScript (~189,500 行) |
| 終端機 UI | React + Ink |
| 執行時 | Node.js >= 20.0.0 |
| 套件管理 | npm workspaces |
| 測試框架 | Vitest 3.2.4 |
| 建置工具 | esbuild |

---

## 主要系統模組

### 1. CLI 套件 (`/packages/cli`)

**入口點:** `index.ts` → `gemini.tsx:main()`

**核心流程:**
```
main()
├─ 載入設定
├─ 解析 CLI 參數
├─ 檢查沙盒需求
├─ 初始化應用元件
├─ 判斷互動 vs 非互動模式
│   ├─ 互動: startInteractiveUI()
│   └─ 非互動: runNonInteractive()
└─ 清理和退出
```

**關鍵檔案:**
- `AppContainer.tsx` (1714 行) - 主要狀態容器
- `InputPrompt.tsx` (1235 行) - 輸入欄位
- `useGeminiStream.ts` (1338 行) - 串流管理

### 2. Core 套件 (`/packages/core`)

**主要組件:**

| 組件 | 檔案 | 用途 |
|------|------|------|
| BaseLlmClient | baseLlmClient.ts | LLM API 呼叫 |
| GeminiChat | geminiChat.ts | 會話管理 |
| CoreToolScheduler | coreToolScheduler.ts | 工具執行 |
| HookSystem | hooks/*.ts | Hook 協調 |
| PolicyEngine | policy/*.ts | 安全政策 |

### 3. 工具系統

**工具數量:** 23+ 內建工具

**工具類別:**
- 檔案工具: ReadFile, WriteFile, Edit
- 搜尋工具: Glob, Grep, RipGrep
- Shell 工具: Shell (PTY/child_process)
- Web 工具: WebFetch, WebSearch
- MCP 工具: 第三方整合
- 記憶工具: MemoryTool
- 智慧編輯: SmartEdit

**三層驗證:**
1. Schema 驗證 (JSON schema)
2. 參數驗證 (業務邏輯)
3. 執行批准 (政策決策)

### 4. Hooks 系統

**事件類型:** 11 種

```typescript
enum HookEventName {
  BeforeTool,           // 工具執行前
  AfterTool,            // 工具執行後
  BeforeAgent,          // 代理提示前
  AfterAgent,           // 代理回應後
  Notification,         // 通知事件
  SessionStart,         // 會話初始化
  SessionEnd,           // 會話終止
  PreCompress,          // 歷史壓縮前
  BeforeModel,          // LLM 呼叫前
  AfterModel,           // LLM 呼叫後
  BeforeToolSelection,  // 工具選擇前
}
```

**聚合策略:**
- OR 邏輯: BeforeTool, AfterTool (任何可阻擋)
- 欄位替換: BeforeModel, AfterModel
- 工具聯合: BeforeToolSelection

### 5. 服務層

**核心服務:** 13 個

| 服務 | 用途 |
|------|------|
| ShellExecutionService | Shell 命令執行 |
| FileDiscoveryService | 檔案發現與過濾 |
| GitService | Git 影子儲存庫 |
| ChatRecordingService | 對話持久化 |
| SessionSummaryService | LLM 摘要生成 |
| ModelConfigService | 模型配置解析 |
| LoopDetectionService | 無限迴圈偵測 |
| SkillManager | SKILL.md 發現 |
| ContextManager | 三層記憶系統 |
| ChatCompressionService | Token 感知壓縮 |

### 6. 配置與政策系統

**設定範圍:**
1. SystemDefaults - 最低優先順序
2. User (~/.gemini/settings.json)
3. Workspace (.gemini/settings.json)
4. System - 最高優先順序

**政策優先順序:**
```
Tier 3: 管理員政策 (3.000-3.999)
Tier 2: 使用者政策 (2.000-2.999)
Tier 1: 預設政策 (1.000-1.999)
```

### 7. 認證與 MCP 系統

**認證方法:**
- OAuth 2.0 (Google 登入)
- API Key (Gemini)
- Vertex AI
- Service Account Impersonation
- Application Default Credentials

**MCP 傳輸:**
- Stdio (子程序)
- SSE (Server-Sent Events)
- HTTP (Streamable)

**Token 儲存:**
- OS Keychain (主要)
- AES-256-GCM 加密檔案 (回退)

### 8. 代理系統

**代理類型:**
- Local: 本地執行
- Remote: A2A 協議

**內建代理:**
- CodebaseInvestigatorAgent
- IntrospectionAgent

**執行特點:**
- 隔離的 ToolRegistry
- YOLO 批准模式
- 60 秒寬限期恢復

### 9. UI 元件

**元件數量:** 120+

**React Contexts:** 11+
- UIStateContext
- UIActionsContext
- KeypressContext
- MouseContext
- ScrollProvider
- VimModeContext
- SessionContext

**主題數量:** 15+

### 10. 測試架構

**測試檔案:** 541 個
**Node 版本:** 20.x, 22.x, 24.x
**沙盒模式:** none, docker, podman

---

## 關鍵設計模式

### 1. 非同步生成器串流
```typescript
async *sendMessageStream(): AsyncGenerator<StreamEvent>
```
- 即時回應處理
- RETRY 事件支援

### 2. 工具執行狀態機
```
validating → scheduled → executing → (success|error|cancelled)
```

### 3. 分層配置
```
System > Workspace > User > SystemDefaults
```

### 4. Hook 管道
```
Event → Planner → Runner → Aggregator → Translator
```

### 5. 政策分層
```
TOML 規則 + 設定規則 → PolicyEngine.check()
```

---

## 安全機制

| 機制 | 描述 |
|------|------|
| 環境淨化 | 移除敏感變數 |
| 工作區信任 | 資料夾信任驗證 |
| PKCE OAuth | 防止授權碼攔截 |
| Token 加密 | AES-256-GCM |
| 政策引擎 | 三層決策 (ALLOW/DENY/ASK) |
| Hook 信任 | 專案 hooks 需要信任 |

---

## 檔案結構

```
gemini-cli/
├── packages/
│   ├── cli/                    # 前端套件 (React/Ink)
│   │   └── src/
│   │       ├── commands/       # CLI 命令
│   │       ├── config/         # 配置載入
│   │       ├── services/       # CLI 服務
│   │       └── ui/             # UI 元件
│   │           ├── components/ # 120+ 元件
│   │           ├── contexts/   # React contexts
│   │           ├── hooks/      # 50+ hooks
│   │           └── themes/     # 主題
│   ├── core/                   # 後端套件
│   │   └── src/
│   │       ├── agents/         # 代理系統
│   │       ├── core/           # 核心邏輯
│   │       ├── hooks/          # Hook 引擎
│   │       ├── mcp/            # MCP 整合
│   │       ├── policy/         # 政策引擎
│   │       ├── services/       # 核心服務
│   │       └── tools/          # 工具實現
│   ├── a2a-server/             # A2A 協議伺服器
│   ├── test-utils/             # 測試工具
│   └── vscode-ide-companion/   # VS Code 擴展
├── integration-tests/          # 整合測試
├── scripts/                    # 建置腳本
└── analysis/                   # 本分析報告
```

---

## 分析報告索引

| 檔案 | 內容 |
|------|------|
| [01-cli-package.md](./01-cli-package.md) | CLI 套件深度分析 |
| [02-core-package.md](./02-core-package.md) | Core 套件深度分析 |
| [03-tools-system.md](./03-tools-system.md) | 工具系統分析 |
| [04-hooks-system.md](./04-hooks-system.md) | Hooks 系統分析 |
| [05-services-layer.md](./05-services-layer.md) | 服務層分析 |
| [06-config-policy.md](./06-config-policy.md) | 配置與政策系統 |
| [07-auth-mcp.md](./07-auth-mcp.md) | 認證與 MCP 系統 |
| [08-agents-system.md](./08-agents-system.md) | 代理系統分析 |
| [09-ui-components.md](./09-ui-components.md) | UI 元件分析 |
| [10-testing.md](./10-testing.md) | 測試架構分析 |

---

## 關鍵指標摘要

| 指標 | 數值 |
|------|------|
| 總程式碼行數 | ~189,500 行 |
| TypeScript 檔案 | 400+ |
| 測試檔案 | 541 |
| 內建工具 | 23+ |
| UI 元件 | 120+ |
| 自訂 Hooks | 50+ |
| React Contexts | 11+ |
| Hook 事件類型 | 11 |
| 核心服務 | 13 |
| 主題 | 15+ |
| 支援的認證方法 | 5 |
| MCP 傳輸類型 | 3 |

---

## 結論

Gemini CLI 是一個**成熟的企業級 AI 代理框架**，具備：

1. **模組化架構** - 清晰的前後端分離
2. **全面安全** - 多層驗證與政策控制
3. **可擴展性** - MCP 協議支援第三方整合
4. **豐富功能** - 23+ 工具、Hook 系統、代理框架
5. **優秀 UX** - React/Ink 終端機 UI、主題支援
6. **測試完善** - 541 個測試檔案、多 Node 版本支援
7. **文檔良好** - 完整的類型定義與程式碼註釋

該專案展示了現代 CLI 應用程式開發的最佳實踐，特別是在 AI 代理和工具整合方面。
