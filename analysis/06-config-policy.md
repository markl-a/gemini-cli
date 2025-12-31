# 配置與政策系統深度分析報告

## 1. 設定載入系統

### 1.1 設定範圍層次結構

**檔案路徑:** `/packages/cli/src/config/settings.ts` (Lines 166-174)

```typescript
export enum SettingScope {
  User = 'User',              // ~/.gemini/settings.json
  Workspace = 'Workspace',    // .gemini/settings.json
  System = 'System',          // /etc/gemini-cli/settings.json
  SystemDefaults = 'SystemDefaults',  // 系統預設值
  Session = 'Session',        // 執行時設定
}
```

**可載入範圍優先順序 (Lines 179-203):**
1. SystemDefaults - 最低優先順序 (基準配置)
2. User - 使用者目錄設定
3. Workspace - 專案特定設定
4. System - 最高優先順序 (管理員覆蓋)

### 1.2 路徑解析

**系統路徑依作業系統 (Lines 141-162):**
- Linux: `/etc/gemini-cli/settings.json`
- macOS: `/Library/Application Support/GeminiCli/settings.json`
- Windows: `C:\ProgramData\gemini-cli\settings.json`
- 使用者路徑: `$HOME/.gemini/settings.json`
- 工作區路徑: `<workspace>/.gemini/settings.json`

### 1.3 設定載入管道

**核心載入函數 (Lines 616-827):**

```
loadSettings(workspaceDir) {
  1. 載入全部 4 個範圍檔案
  2. 檢查是否需要遷移 (V1 → V2)
  3. 如需要則套用遷移
  4. 使用 Zod schema 驗證
  5. 解析環境變數
  6. 檢查工作區信任狀態
  7. 從 .env 檔案載入環境變數
  8. 根據信任狀態合併設定
  9. 返回 LoadedSettings
}
```

---

## 2. 設定 Schema 定義

**檔案路徑:** `/packages/cli/src/config/settingsSchema.ts` (Lines 140-1701)

### 2.1 主要設定類別

| 類別 | 行數 | 用途 |
|------|------|------|
| `general` | 157-309 | 核心行為設定 |
| `model` | 645-714 | LLM 配置 |
| `tools` | 884-1087 | 工具執行設定 |
| `context` | 768-882 | 檔案包含與記憶發現 |
| `security` | 1147-1286 | 認證、資料夾信任 |
| `ui` | 336-581 | 主題、顯示選項 |
| `hooks` | 1536-1700 | Hook 系統配置 |
| `extensions` | 1478-1511 | 擴展管理 |
| `experimental` | 1339-1475 | 代理功能、技能 |

### 2.2 合併策略

```typescript
export enum MergeStrategy {
  REPLACE = 'replace',           // 預設: 後值覆蓋
  CONCAT = 'concat',             // 陣列連接
  UNION = 'union',               // 陣列合併唯一值
  SHALLOW_MERGE = 'shallow_merge' // 物件淺層合併
}
```

**策略應用範例:**
- `context.includeDirectories`: `CONCAT` (Line 818)
- `tools.exclude`: `UNION` (Line 1015)
- `hooks.*`: `CONCAT` (Line 1571, 1583)
- `mcpServers`: `SHALLOW_MERGE` (Line 150)

---

## 3. 環境變數處理

**檔案路徑:** `/packages/cli/src/utils/envVarResolver.ts` (Lines 1-126)

### 3.1 解析模式

```
模式: $VAR_NAME 或 ${VAR_NAME}
正規表達式: /\$(?:(\w+)|{([^}]+)})/g
回退: 未定義的變數保持不變
```

### 3.2 物件解析 (Lines 56-126)

- 使用 WeakSet 進行循環參考偵測
- 保留原始類型 (null, boolean, number)
- 遞迴處理陣列和巢狀物件
- 防止原型污染 (Line 33): 跳過 `__proto__`

### 3.3 環境檔案載入 (Lines 519-610)

**搜尋演算法:**
1. 從目前工作目錄開始
2. 向上遍歷目錄樹尋找:
   - `.gemini/.env` (專案特定)
   - `.env` (標準)
3. 如果到達 home 目錄，檢查:
   - `~/.gemini/.env`
   - `~/.env`

---

## 4. 設定合併系統

**檔案路徑:** `/packages/cli/src/utils/deepMerge.ts` (Lines 1-95)

### 合併演算法 (Lines 24-79)

```
1. SHALLOW_MERGE (Lines 44-50):
   target[key] = { ...obj1, ...obj2 }

2. CONCAT (Lines 55-57):
   target[key].concat(srcArray)

3. UNION (Lines 59-61):
   [...new Set(array1.concat(array2))]

4. REPLACE (預設) (Lines 75-76):
   純量值直接賦值，巢狀物件遞迴合併
```

### 合併順序 (Lines 426-449)

```
mergeSettings(system, systemDefaults, user, workspace, isTrusted) {
  從空物件 {} 開始
  1. 合併 systemDefaults
  2. 合併 user
  3. 合併 workspace (如果信任) 或 {} (如果不信任)
  4. 合併 system (覆蓋所有)
}
```

---

## 5. V1 到 V2 設定遷移

**檔案路徑:** `/packages/cli/src/config/settings.ts` (Lines 69-425)

### 遷移映射 (Lines 71-139)

將 70+ V1 扁平鍵映射到 V2 巢狀路徑：
```
V1 Key              → V2 Path
accessibility       → ui.accessibility
allowedTools        → tools.allowed
autoAccept          → tools.autoAccept
checkpointing       → general.checkpointing
folderTrust         → security.folderTrust.enabled
theme               → ui.theme
```

### 遷移邏輯 (Lines 293-356)

1. 使用 `setNestedProperty()` 創建巢狀結構
2. 保留 mcpServers 在頂層
3. 智慧合併未識別的鍵
4. 衝突時使用策略感知合併

---

## 6. 政策引擎系統

### 6.1 核心政策類型

**檔案路徑:** `/packages/core/src/policy/types.ts` (Lines 1-248)

```typescript
enum PolicyDecision {
  ALLOW = 'allow',      // 無需確認執行
  DENY = 'deny',        // 阻擋執行
  ASK_USER = 'ask_user' // 提示使用者
}

enum ApprovalMode {
  DEFAULT = 'default',    // 正常互動模式
  AUTO_EDIT = 'autoEdit', // 自動批准編輯操作
  YOLO = 'yolo'          // 自動批准所有 (危險)
}
```

### 6.2 政策規則介面 (Lines 97-126)

```typescript
interface PolicyRule {
  toolName?: string;        // "run_shell_command"
  argsPattern?: RegExp;     // 參數匹配模式
  decision: PolicyDecision; // 必要決策
  priority?: number;        // 排序順序
  modes?: ApprovalMode[];   // 適用模式
}
```

---

## 7. 政策引擎實現

**檔案路徑:** `/packages/core/src/policy/policy-engine.ts` (Lines 102-444)

### 引擎結構 (Lines 112-127)

```typescript
class PolicyEngine {
  private rules: PolicyRule[];           // 按優先順序排序
  private checkers: SafetyCheckerRule[];
  private hookCheckers: HookCheckerRule[];
  private defaultDecision = ASK_USER;
  private nonInteractive = false;
  private allowHooks = true;
  private approvalMode = ApprovalMode.DEFAULT;
}
```

### 規則匹配 (Lines 29-80)

```
ruleMatches(rule, toolCall, args, serverName, approvalMode) {
  1. 檢查 approvalMode 過濾器
  2. 檢查 toolName:
     - 精確匹配: "run_shell_command"
     - 萬用字元模式: "serverName__*"
  3. 檢查 argsPattern 正規表達式
  4. 返回匹配決策
}
```

### 工具呼叫評估 (Lines 147-310)

```
check(toolCall, serverName):
  1. 字串化參數
  2. 找到第一個匹配規則
  3. Shell 命令: 遞迴檢查子命令
  4. 無匹配規則使用 defaultDecision
  5. 執行安全檢查器
  6. 套用非互動模式
  7. 返回決策 + 匹配規則
```

---

## 8. TOML 政策規則

**檔案路徑:** `/packages/core/src/policy/toml-loader.ts` (Lines 1-524)

### 8.1 TOML Schema (Lines 23-47)

```toml
[[rule]]
toolName = "run_shell_command"
decision = "allow" or "deny" or "ask_user"
priority = 50                       # 0-999
argsPattern = "regex"
modes = ["yolo"]
commandPrefix = "git"
commandRegex = "git push.*"
```

### 8.2 優先順序分層 (Lines 208-210)

```
transformPriority(priority, tier) {
  return tier + priority / 1000
}

層級:
  1: 預設政策 (1.000 - 1.999)
  2: 使用者政策 (2.000 - 2.999)
  3: 管理員政策 (3.000 - 3.999)
```

### 優先順序範圍

```
Tier 1 (預設 TOML):
  1.010: 寫入工具 (ask_user)
  1.015: Auto-edit 覆蓋
  1.050: 唯讀工具 (allow)
  1.999: YOLO allow-all

Tier 2 (使用者設定/動態):
  2.1:  MCP 伺服器允許列表
  2.2:  MCP 伺服器 trust=true
  2.3:  工具允許列表
  2.4:  工具排除列表
  2.9:  MCP 伺服器排除列表
  2.95: 使用者「總是允許」選擇
```

---

## 9. 工作區信任系統

**檔案路徑:** `/packages/cli/src/config/trustedFolders.ts` (Lines 1-80)

### 信任級別 (Lines 33-44)

```typescript
enum TrustLevel {
  TRUST_FOLDER = 'TRUST_FOLDER',   // 僅此資料夾
  TRUST_PARENT = 'TRUST_PARENT',   // 此 + 父資料夾
  DO_NOT_TRUST = 'DO_NOT_TRUST',   // 明確不信任
}
```

### 信任檔案儲存 (Lines 20-31)

```
檔案: ~/.gemini/trustedFolders.json
內容: {
  "/path/to/project": "TRUST_FOLDER",
  "/path/to/workspace": "DO_NOT_TRUST"
}
```

---

## 10. 執行流程圖

### 應用程式啟動流程

```
main()
  ↓
loadSettings(workspaceDir)
  ├─ 載入系統設定
  ├─ 載入系統預設值
  ├─ 載入使用者設定
  ├─ 偵測到 V1 則遷移
  ├─ 使用 Zod 驗證
  ├─ 解析環境變數
  ├─ 檢查工作區信任
  ├─ 載入 .env 檔案
  └─ 返回 LoadedSettings

LoadedSettings.merged
  ↓
createPolicyEngineConfig(merged.mcp, merged.tools)
  ├─ 載入 TOML 檔案
  ├─ 從設定創建規則
  ├─ 按優先順序排序
  └─ 返回 PolicyEngineConfig

new PolicyEngine(config)
  ├─ 排序規則
  └─ 準備評估
```

### 工具執行政策檢查

```
toolExecution("run_shell_command", {command: "npm test"})
  ↓
policyEngine.check(...)
  ├─ 對每個規則 (按優先順序):
  │  ├─ 檢查模式過濾器
  │  ├─ 檢查 toolName 匹配
  │  ├─ 檢查 argsPattern
  │  └─ 匹配則處理 shell 子命令
  │
  ├─ 無匹配: 使用 defaultDecision (ASK_USER)
  │
  ├─ 執行安全檢查器
  │
  └─ 套用非互動模式
     (ASK_USER → DENY 如果非互動)

結果:
  ├─ ALLOW → 立即執行
  ├─ DENY → 阻擋並顯示錯誤
  └─ ASK_USER → 提示使用者 (互動) 或 DENY (非互動)
```

---

## 關鍵設計模式

### 1. 分層配置與信任邊界

```
信任狀態
  ├─ System (始終載入，最高優先順序)
  ├─ User (始終載入)
  ├─ Workspace (條件載入，最低優先順序)
  └─ SystemDefaults (基準，最低優先順序)

信任檢查:
  - 不信任: 忽略工作區設定
  - 信任: 包含工作區設定
```

### 2. 原子政策檔案更新

```
1. 寫入暫時檔案: ~/.gemini/policies/auto-saved.toml.tmp
2. 原子重命名: .tmp → .toml
3. 優點: 讀取者看不到部分寫入
```

### 3. 循環參考保護

```
WeakSet visited 追蹤物件參考
  ├─ 偵測循環: A → B → A
  ├─ 打破循環: 返回淺複製
  └─ 防止: 無限遞迴、堆疊溢出
```
