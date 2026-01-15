# Gemini CLI 測試架構深度分析報告

## 概述

本報告深入分析 Gemini CLI 專案的測試架構，涵蓋測試框架配置、檔案組織、Mock 策略、CI/CD 整合等核心面向。專案採用 **Vitest 3.2.4** 作為測試框架，配合 **MSW (Mock Service Worker)** 進行 HTTP 模擬，並建立了完整的整合測試輔助框架 (`TestRig`)。

---

## 1. 測試框架設置 (Vitest 配置)

### 1.1 Vitest 核心配置對比

專案包含 **6 個獨立的 Vitest 配置檔案**，針對不同套件和測試類型進行優化：

| 配置檔案 | 用途 | 超時 | 重試 | 特殊設置 |
|----------|------|------|------|----------|
| `/packages/cli/vitest.config.ts` | CLI 套件單元測試 | 預設 (5s) | 0 | React 環境、別名配置 |
| `/packages/core/vitest.config.ts` | Core 套件單元測試 | 30000ms | 0 | silent: true |
| `/packages/a2a-server/vitest.config.ts` | A2A Server 測試 | 預設 | 0 | 標準配置 |
| `/packages/test-utils/vitest.config.ts` | 測試工具測試 | 預設 | 0 | 標準配置 |
| `/integration-tests/vitest.config.ts` | 端對端整合測試 | 300000ms (5分鐘) | 2 | globalSetup |
| `/scripts/tests/vitest.config.ts` | 腳本測試 | 預設 | 0 | 標準配置 |

### 1.2 CLI 套件配置詳解

**檔案路徑**: `/packages/cli/vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config';
import { fileURLToPath } from 'node:url';
import * as path from 'node:path';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export default defineConfig({
  resolve: {
    conditions: ['test'],  // 啟用測試條件導入
  },
  test: {
    // 測試檔案匹配模式
    include: ['**/*.{test,spec}.{js,ts,jsx,tsx}', 'config.test.ts'],
    exclude: ['**/node_modules/**', '**/dist/**', '**/cypress/**'],

    // 環境設置
    environment: 'node',
    globals: true,  // describe, it, expect 全域可用

    // 報告輸出
    reporters: ['default', 'junit'],
    outputFile: {
      junit: 'junit.xml',  // CI 可讀取的 JUnit 格式
    },

    // React 相關
    alias: {
      react: path.resolve(__dirname, '../../node_modules/react'),
    },

    // 設置檔案
    setupFiles: ['./test-setup.ts'],

    // 覆蓋率配置
    coverage: {
      enabled: true,
      provider: 'v8',
      reportsDirectory: './coverage',
      include: ['src/**/*'],
      reporter: [
        ['text', { file: 'full-text-summary.txt' }],
        'html',
        'json',
        'lcov',
        'cobertura',
        ['json-summary', { outputFile: 'coverage-summary.json' }],
      ],
    },

    // 執行緒池配置
    poolOptions: {
      threads: {
        minThreads: 8,
        maxThreads: 16,
      },
    },

    // 依賴內聯
    server: {
      deps: {
        inline: [/@google\/gemini-cli-core/],
      },
    },
  },
});
```

### 1.3 整合測試配置詳解

**檔案路徑**: `/integration-tests/vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    testTimeout: 300000,        // 5 分鐘超時（整合測試需要較長時間）
    globalSetup: './globalSetup.ts',  // 全域設置/清理
    reporters: ['default'],
    include: ['**/*.test.ts'],
    retry: 2,                   // 失敗自動重試 2 次
    fileParallelism: true,      // 檔案層級並行
    poolOptions: {
      threads: {
        minThreads: 8,
        maxThreads: 16,
      },
    },
  },
});
```

### 1.4 測試腳本命令

**根目錄 `package.json` 測試腳本**:

```json
{
  "scripts": {
    // 基本測試
    "test": "npm run test --workspaces --if-present",
    "test:ci": "npm run test:ci --workspaces --if-present && npm run test:scripts",
    "test:scripts": "vitest run --config ./scripts/tests/vitest.config.ts",

    // 端對端測試
    "test:e2e": "cross-env VERBOSE=true KEEP_OUTPUT=true npm run test:integration:sandbox:none",

    // 整合測試（多種沙盒模式）
    "test:integration:all": "npm run test:integration:sandbox:none && npm run test:integration:sandbox:docker && npm run test:integration:sandbox:podman",
    "test:integration:sandbox:none": "cross-env GEMINI_SANDBOX=false vitest run --root ./integration-tests",
    "test:integration:sandbox:docker": "cross-env GEMINI_SANDBOX=docker npm run build:sandbox && cross-env GEMINI_SANDBOX=docker vitest run --root ./integration-tests",
    "test:integration:sandbox:podman": "cross-env GEMINI_SANDBOX=podman vitest run --root ./integration-tests",

    // 穩定性測試
    "deflake": "node scripts/deflake.js",
    "deflake:test:integration:sandbox:none": "npm run deflake -- --command=\"npm run test:integration:sandbox:none -- --retry=0\"",
    "deflake:test:integration:sandbox:docker": "npm run deflake -- --command=\"npm run test:integration:sandbox:docker -- --retry=0\""
  }
}
```

---

## 2. 測試檔案組織與模式

### 2.1 測試檔案統計

```
總測試檔案數量: 541 個
├── packages/cli/        ~420 個測試檔案
├── packages/core/       ~80 個測試檔案
├── packages/a2a-server/ ~15 個測試檔案
├── packages/test-utils/ ~5 個測試檔案
├── integration-tests/   24 個測試檔案
└── scripts/tests/       ~3 個測試檔案
```

### 2.2 目錄結構

```
gemini-cli/
├── packages/
│   ├── cli/
│   │   ├── test-setup.ts                    # CLI 測試設置
│   │   ├── vitest.config.ts                 # Vitest 配置
│   │   └── src/
│   │       ├── config/
│   │       │   ├── config.test.ts           # 單元測試
│   │       │   ├── config.integration.test.ts # 整合測試
│   │       │   ├── settings.test.ts
│   │       │   ├── settingsSchema.test.ts
│   │       │   ├── trustedFolders.test.ts
│   │       │   └── extensions/
│   │       │       ├── consent.test.ts
│   │       │       ├── extensionEnablement.test.ts
│   │       │       ├── github.test.ts
│   │       │       └── storage.test.ts
│   │       ├── commands/
│   │       │   ├── extensions.test.tsx
│   │       │   ├── mcp.test.ts
│   │       │   └── extensions/
│   │       │       ├── install.test.ts
│   │       │       ├── uninstall.test.ts
│   │       │       ├── enable.test.ts
│   │       │       ├── disable.test.ts
│   │       │       ├── link.test.ts
│   │       │       └── validate.test.ts
│   │       ├── services/
│   │       │   ├── CommandService.test.ts
│   │       │   ├── FileCommandLoader.test.ts
│   │       │   ├── BuiltinCommandLoader.test.ts
│   │       │   └── prompt-processors/
│   │       │       ├── atFileProcessor.test.ts
│   │       │       ├── shellProcessor.test.ts
│   │       │       └── argumentProcessor.test.ts
│   │       ├── ui/
│   │       │   ├── App.test.tsx
│   │       │   ├── AppContainer.test.tsx
│   │       │   ├── auth/
│   │       │   │   ├── AuthDialog.test.tsx
│   │       │   │   ├── ApiAuthDialog.test.tsx
│   │       │   │   └── useAuth.test.tsx
│   │       │   ├── commands/
│   │       │   │   ├── aboutCommand.test.ts
│   │       │   │   ├── authCommand.test.ts
│   │       │   │   ├── clearCommand.test.ts
│   │       │   │   └── ... (40+ 命令測試)
│   │       │   └── utils/
│   │       │       ├── updateCheck.test.ts
│   │       │       ├── ui-sizing.test.ts
│   │       │       └── commandUtils.test.ts
│   │       └── test-utils/
│   │           ├── render.tsx               # React 渲染工具
│   │           ├── render.test.tsx
│   │           ├── async.ts                 # 非同步工具
│   │           ├── customMatchers.ts        # 自訂匹配器
│   │           └── mockCommandContext.test.ts
│   ├── core/
│   │   ├── test-setup.ts
│   │   ├── vitest.config.ts
│   │   └── src/
│   │       ├── config/
│   │       │   └── config.test.ts
│   │       ├── telemetry/
│   │       │   └── clearcut-logger/
│   │       │       └── clearcut-logger.test.ts
│   │       ├── mocks/
│   │       │   └── msw.ts                   # MSW 伺服器設置
│   │       └── test-utils/
│   │           ├── mock-tool.ts             # Mock 工具類
│   │           └── mock-message-bus.ts
│   └── test-utils/
│       └── src/
│           └── file-system-test-helpers.ts  # 檔案系統輔助
├── integration-tests/
│   ├── vitest.config.ts
│   ├── globalSetup.ts                       # 全域設置/清理
│   ├── test-helper.ts                       # TestRig 框架 (1118 行)
│   ├── file-system.test.ts
│   ├── file-system-interactive.test.ts
│   ├── hooks-system.test.ts
│   ├── hooks-agent-flow.test.ts
│   ├── telemetry.test.ts
│   ├── extensions-install.test.ts
│   ├── run_shell_command.test.ts
│   ├── write_file.test.ts
│   ├── replace.test.ts
│   ├── read_many_files.test.ts
│   ├── ctrl-c-exit.test.ts
│   ├── json-output.test.ts
│   ├── simple-mcp-server.test.ts
│   └── ... (更多整合測試)
└── scripts/
    └── tests/
        ├── vitest.config.ts
        └── generate-settings-schema.test.ts
```

### 2.3 命名慣例

| 測試類型 | 檔案命名模式 | 位置 | 範例 |
|----------|--------------|------|------|
| 單元測試 | `*.test.ts` | `packages/*/src/**` | `config.test.ts` |
| React 元件測試 | `*.test.tsx` | `packages/*/src/**` | `App.test.tsx` |
| 混合整合測試 | `*.integration.test.ts` | `packages/*/src/**` | `config.integration.test.ts` |
| 端對端整合測試 | `*.test.ts` | `integration-tests/` | `file-system.test.ts` |
| 互動式測試 | `*-interactive.test.ts` | `integration-tests/` | `file-system-interactive.test.ts` |
| 規格測試 | `*.spec.ts` | 任意位置 | (少用) |

---

## 3. 測試工具與輔助函數

### 3.1 全域測試設置檔案

#### CLI test-setup.ts (65 行)

**檔案路徑**: `/packages/cli/test-setup.ts`

```typescript
import { vi, beforeEach, afterEach } from 'vitest';
import { format } from 'node:util';

// 啟用 React 測試環境
global.IS_REACT_ACT_ENVIRONMENT = true;

// 確保一致的主題行為
if (process.env.NO_COLOR !== undefined) {
  delete process.env.NO_COLOR;
}

// 導入自訂匹配器
import './src/test-utils/customMatchers.js';

let consoleErrorSpy: vi.SpyInstance;
let actWarnings: Array<{ message: string; stack: string }> = [];

beforeEach(() => {
  actWarnings = [];
  // 監視 console.error 以捕捉 React act() 警告
  consoleErrorSpy = vi.spyOn(console, 'error').mockImplementation((...args) => {
    const firstArg = args[0];
    if (
      typeof firstArg === 'string' &&
      firstArg.includes('was not wrapped in act(...)')
    ) {
      // 提取有意義的堆疊追蹤
      const stackLines = (new Error().stack || '').split('\n');
      let lastReactFrameIndex = -1;

      for (let i = 0; i < stackLines.length; i++) {
        if (stackLines[i].includes('react-reconciler')) {
          lastReactFrameIndex = i;
        }
      }

      const relevantStack =
        lastReactFrameIndex !== -1
          ? stackLines.slice(lastReactFrameIndex + 1).join('\n')
          : stackLines.slice(1).join('\n');

      actWarnings.push({
        message: format(...args),
        stack: relevantStack,
      });
    }
  });
});

afterEach(() => {
  consoleErrorSpy.mockRestore();

  // 如果有 act() 警告，測試失敗
  if (actWarnings.length > 0) {
    const messages = actWarnings
      .map(({ message, stack }) => `${message}\n${stack}`)
      .join('\n\n');
    throw new Error(`Failing test due to "act(...)" warnings:\n${messages}`);
  }
});
```

**核心功能**:
- 自動偵測並捕捉 React `act()` 警告
- 測試失敗時提供清晰的堆疊追蹤
- 確保一致的環境行為

#### Core test-setup.ts (16 行)

**檔案路徑**: `/packages/core/test-setup.ts`

```typescript
// 確保一致的主題行為
if (process.env.NO_COLOR !== undefined) {
  delete process.env.NO_COLOR;
}

import { setSimulate429 } from './src/utils/testUtils.js';

// 全域停用 429 模擬
setSimulate429(false);
```

#### 整合測試 globalSetup.ts (84 行)

**檔案路徑**: `/integration-tests/globalSetup.ts`

```typescript
// 確保一致的主題行為
if (process.env['NO_COLOR'] !== undefined) {
  delete process.env['NO_COLOR'];
}

import { mkdir, readdir, rm } from 'node:fs/promises';
import { join, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
import { canUseRipgrep } from '../packages/core/src/tools/ripGrep.js';

const __dirname = dirname(fileURLToPath(import.meta.url));
const rootDir = join(__dirname, '..');
const integrationTestsDir = join(rootDir, '.integration-tests');
let runDir = '';

export async function setup() {
  // 創建唯一的測試執行目錄
  runDir = join(integrationTestsDir, `${Date.now()}`);
  await mkdir(runDir, { recursive: true });

  // 設置 HOME 環境變數以隔離測試
  process.env['HOME'] = runDir;
  if (process.platform === 'win32') {
    process.env['USERPROFILE'] = runDir;
  }

  // 明確設置配置目錄
  process.env['GEMINI_CONFIG_DIR'] = join(runDir, '.gemini');

  // 預先下載 ripgrep 以避免平行測試競爭
  const available = await canUseRipgrep();
  if (!available) {
    throw new Error('Failed to download ripgrep binary');
  }

  // 清理舊測試執行（保留最新 5 個用於調試）
  try {
    const testRuns = await readdir(integrationTestsDir);
    if (testRuns.length > 5) {
      const oldRuns = testRuns.sort().slice(0, testRuns.length - 5);
      await Promise.all(
        oldRuns.map((oldRun) =>
          rm(join(integrationTestsDir, oldRun), {
            recursive: true,
            force: true,
          }),
        ),
      );
    }
  } catch (e) {
    console.error('Error cleaning up old test runs:', e);
  }

  // 設置測試專用環境變數
  process.env['INTEGRATION_TEST_FILE_DIR'] = runDir;
  process.env['GEMINI_CLI_INTEGRATION_TEST'] = 'true';
  process.env['GEMINI_FORCE_FILE_STORAGE'] = 'true';  // 避免 CI 中的 keychain 問題
  process.env['TELEMETRY_LOG_FILE'] = join(runDir, 'telemetry.log');
  process.env['VERBOSE'] = process.env['VERBOSE'] ?? 'false';

  if (process.env['KEEP_OUTPUT']) {
    console.log(`Keeping output for test run in: ${runDir}`);
  }

  console.log(`\nIntegration test output directory: ${runDir}`);
}

export async function teardown() {
  // 清理測試目錄（除非設置 KEEP_OUTPUT）
  if (process.env['KEEP_OUTPUT'] !== 'true' && runDir) {
    try {
      await rm(runDir, { recursive: true, force: true });
    } catch (e) {
      console.warn('Failed to clean up test run directory:', e);
    }
  }
}
```

**流程圖**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        globalSetup.ts 執行流程                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐         │
│  │ 清理 NO_COLOR │────>│ 創建執行目錄 │────>│ 設置 HOME    │         │
│  │ 環境變數     │     │ .integration │     │ 環境變數     │         │
│  │              │     │ -tests/{ts}/ │     │              │         │
│  └──────────────┘     └──────────────┘     └──────────────┘         │
│                                                    │                 │
│                                                    v                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐         │
│  │ 設置測試     │<────│ 清理舊執行   │<────│ 下載 ripgrep │         │
│  │ 環境變數     │     │ (保留最新5個)│     │ 二進位檔     │         │
│  └──────────────┘     └──────────────┘     └──────────────┘         │
│         │                                                            │
│         v                                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      測試執行                                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│         │                                                            │
│         v                                                            │
│  ┌──────────────┐                                                   │
│  │ teardown()   │                                                   │
│  │ 清理測試目錄 │                                                   │
│  │ (除非 KEEP_  │                                                   │
│  │  OUTPUT=true)│                                                   │
│  └──────────────┘                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 檔案系統測試輔助函數

**檔案路徑**: `/packages/test-utils/src/file-system-test-helpers.ts`

```typescript
import * as fs from 'node:fs/promises';
import * as path from 'node:path';
import * as os from 'node:os';

/**
 * 定義虛擬檔案系統結構
 *
 * @example
 * // 範例 1: 簡單檔案和目錄
 * const structure1 = {
 *   'file1.txt': 'Hello, world!',
 *   'empty-dir': [],
 *   'src': {
 *     'main.js': '// Main application file',
 *     'utils.ts': '// Utility functions',
 *   },
 * };
 *
 * @example
 * // 範例 2: 巢狀目錄和陣列中的空檔案
 * const structure2 = {
 *   'config.json': '{ "port": 3000 }',
 *   'data': [
 *     'users.csv',
 *     'products.json',
 *     {
 *       'logs': ['error.log', 'access.log'],
 *     },
 *   ],
 * };
 */
export type FileSystemStructure = {
  [name: string]:
    | string                              // 檔案內容
    | FileSystemStructure                 // 子目錄
    | Array<string | FileSystemStructure>; // 帶檔案的目錄
};

/**
 * 遞迴創建檔案和目錄
 */
async function create(dir: string, structure: FileSystemStructure) {
  for (const [name, content] of Object.entries(structure)) {
    const newPath = path.join(dir, name);
    if (typeof content === 'string') {
      // 字串 = 檔案內容
      await fs.writeFile(newPath, content);
    } else if (Array.isArray(content)) {
      // 陣列 = 目錄中的檔案/子目錄
      await fs.mkdir(newPath, { recursive: true });
      for (const item of content) {
        if (typeof item === 'string') {
          await fs.writeFile(path.join(newPath, item), '');
        } else {
          await create(newPath, item);
        }
      }
    } else if (typeof content === 'object' && content !== null) {
      // 物件 = 子目錄
      await fs.mkdir(newPath, { recursive: true });
      await create(newPath, content);
    }
  }
}

/**
 * 創建臨時目錄並填充檔案結構
 */
export async function createTmpDir(
  structure: FileSystemStructure,
): Promise<string> {
  const tmpDir = await fs.mkdtemp(path.join(os.tmpdir(), 'gemini-cli-test-'));
  await create(tmpDir, structure);
  return tmpDir;
}

/**
 * 清理臨時目錄
 */
export async function cleanupTmpDir(dir: string) {
  await fs.rm(dir, { recursive: true, force: true });
}
```

**使用範例**:

```typescript
import { createTmpDir, cleanupTmpDir } from '@google/gemini-cli-test-utils';

describe('My Feature', () => {
  let tmpDir: string;

  beforeEach(async () => {
    tmpDir = await createTmpDir({
      'config.json': '{ "debug": true }',
      'src': {
        'index.ts': 'export const main = () => {}',
        'utils': {
          'helper.ts': '// Helper functions',
        },
      },
      'data': ['file1.txt', 'file2.txt'],
    });
  });

  afterEach(async () => {
    await cleanupTmpDir(tmpDir);
  });

  it('should process files', async () => {
    // 測試使用 tmpDir
  });
});
```

### 3.3 React 元件測試工具

**檔案路徑**: `/packages/cli/src/test-utils/render.tsx`

```typescript
import { render as inkRender } from 'ink-testing-library';
import { Box } from 'ink';
import type React from 'react';
import { vi } from 'vitest';
import { act, useState } from 'react';
import { LoadedSettings, type Settings } from '../config/settings.js';
// ... 其他 context 導入

/**
 * 包裝 ink-testing-library 的 render，確保 act() 被調用
 */
export const render = (
  tree: React.ReactElement,
  terminalWidth?: number,
): ReturnType<typeof inkRender> => {
  let renderResult: ReturnType<typeof inkRender>;

  // 使用 act() 包裝渲染
  act(() => {
    renderResult = inkRender(tree);
  });

  // 如果指定了終端寬度，覆蓋 stdout.columns
  if (terminalWidth !== undefined && renderResult?.stdout) {
    Object.defineProperty(renderResult.stdout, 'columns', {
      get: () => terminalWidth,
      configurable: true,
    });

    // 觸發重新渲染以更新終端寬度
    act(() => {
      renderResult.rerender(tree);
    });
  }

  // 包裝 unmount 和 rerender 方法
  const originalUnmount = renderResult.unmount;
  const originalRerender = renderResult.rerender;

  return {
    ...renderResult,
    unmount: () => {
      act(() => {
        originalUnmount();
      });
    },
    rerender: (newTree: React.ReactElement) => {
      act(() => {
        originalRerender(newTree);
      });
    },
  };
};

/**
 * 模擬滑鼠點擊
 */
export const simulateClick = async (
  stdin: ReturnType<typeof inkRender>['stdin'],
  col: number,
  row: number,
  button: 0 | 1 | 2 = 0, // 0=左鍵, 1=中鍵, 2=右鍵
) => {
  const mouseEventString = `\x1b[<${button};${col};${row}M`;
  await act(async () => {
    stdin.write(mouseEventString);
  });
};

/**
 * 預設 mock Config
 */
const mockConfig = {
  getModel: () => 'gemini-pro',
  getTargetDir: () => '/Users/test/project/foo/bar/...',
  getDebugMode: () => false,
  isTrustedFolder: () => true,
  getIdeMode: () => false,
  getEnableInteractiveShell: () => true,
  getPreviewFeatures: () => false,
};

/**
 * 預設 mock Settings
 */
export const mockSettings = new LoadedSettings(
  { path: '', settings: {}, originalSettings: {} },
  { path: '', settings: {}, originalSettings: {} },
  { path: '', settings: {}, originalSettings: {} },
  { path: '', settings: {}, originalSettings: {} },
  true,
  new Set(),
);

/**
 * 創建自訂 mock Settings
 */
export const createMockSettings = (
  overrides: Partial<Settings>,
): LoadedSettings => {
  const settings = overrides as Settings;
  return new LoadedSettings(
    { path: '', settings: {}, originalSettings: {} },
    { path: '', settings: {}, originalSettings: {} },
    { path: '', settings, originalSettings: settings },
    { path: '', settings: {}, originalSettings: {} },
    true,
    new Set(),
  );
};

/**
 * 預設 mock UIActions
 */
const mockUIActions: UIActions = {
  handleThemeSelect: vi.fn(),
  closeThemeDialog: vi.fn(),
  handleAuthSelect: vi.fn(),
  setAuthState: vi.fn(),
  onAuthError: vi.fn(),
  handleEditorSelect: vi.fn(),
  exitEditorDialog: vi.fn(),
  exitPrivacyNotice: vi.fn(),
  closeSettingsDialog: vi.fn(),
  closeModelDialog: vi.fn(),
  openPermissionsDialog: vi.fn(),
  // ... 更多 mock 方法
};

/**
 * 帶完整 Provider 的渲染
 */
export const renderWithProviders = (
  component: React.ReactElement,
  {
    shellFocus = true,
    settings = mockSettings,
    uiState: providedUiState,
    width,
    mouseEventsEnabled = false,
    config = configProxy as unknown as Config,
    useAlternateBuffer = true,
    uiActions,
  }: {
    shellFocus?: boolean;
    settings?: LoadedSettings;
    uiState?: Partial<UIState>;
    width?: number;
    mouseEventsEnabled?: boolean;
    config?: Config;
    useAlternateBuffer?: boolean;
    uiActions?: Partial<UIActions>;
  } = {},
): ReturnType<typeof render> & { simulateClick: typeof simulateClick } => {
  // 組合 UIState
  const baseState: UIState = new Proxy(
    { ...baseMockUiState, ...providedUiState },
    { /* Proxy 處理 */ }
  ) as UIState;

  const terminalWidth = width ?? baseState.terminalWidth;
  const finalUIActions = { ...mockUIActions, ...uiActions };

  const renderResult = render(
    <ConfigContext.Provider value={config}>
      <SettingsContext.Provider value={finalSettings}>
        <UIStateContext.Provider value={finalUiState}>
          <VimModeProvider settings={finalSettings}>
            <ShellFocusContext.Provider value={shellFocus}>
              <StreamingContext.Provider value={finalUiState.streamingState}>
                <UIActionsContext.Provider value={finalUIActions}>
                  <KeypressProvider>
                    <MouseProvider mouseEventsEnabled={mouseEventsEnabled}>
                      <ScrollProvider>
                        <Box width={terminalWidth}>
                          {component}
                        </Box>
                      </ScrollProvider>
                    </MouseProvider>
                  </KeypressProvider>
                </UIActionsContext.Provider>
              </StreamingContext.Provider>
            </ShellFocusContext.Provider>
          </VimModeProvider>
        </UIStateContext.Provider>
      </SettingsContext.Provider>
    </ConfigContext.Provider>,
    terminalWidth,
  );

  return { ...renderResult, simulateClick };
};

/**
 * Hook 測試工具
 */
export function renderHook<Result, Props>(
  renderCallback: (props: Props) => Result,
  options?: {
    initialProps?: Props;
    wrapper?: React.ComponentType<{ children: React.ReactNode }>;
  },
): {
  result: { current: Result };
  rerender: (props?: Props) => void;
  unmount: () => void;
} {
  const result = { current: undefined as unknown as Result };
  // ... 實作細節
  return { result, rerender, unmount };
}
```

### 3.4 自訂非同步等待函數

**檔案路徑**: `/packages/cli/src/test-utils/async.ts`

```typescript
import { act } from 'react';

/**
 * 自訂 waitFor 實作
 *
 * 解決 vitest 的 waitFor 不包裝 act() 的問題
 */
export async function waitFor(
  assertion: () => void,
  { timeout = 1000, interval = 50 } = {},
): Promise<void> {
  const startTime = Date.now();

  while (true) {
    try {
      assertion();
      return;
    } catch (error) {
      if (Date.now() - startTime > timeout) {
        throw error;
      }

      // 使用 act() 包裝等待
      await act(async () => {
        await new Promise((resolve) => setTimeout(resolve, interval));
      });
    }
  }
}
```

---

## 4. Mock 策略

### 4.1 vi.mock - 完整模組替換

#### 基本用法

```typescript
// 完全替換模組
vi.mock('./trustedFolders.js', () => ({
  isWorkspaceTrusted: vi.fn(() => ({ isTrusted: true, source: 'file' })),
}));
```

#### 保留原始實作的部分替換

```typescript
vi.mock('fs', async (importOriginal) => {
  const actualFs = await importOriginal<typeof import('fs')>();
  return {
    ...actualFs,  // 保留所有原始函數
    mkdirSync: vi.fn(),
    writeFileSync: vi.fn(),
    existsSync: vi.fn((p) => mockPaths.has(p.toString())),
  };
});
```

#### 替換 OS 模組

```typescript
vi.mock('os', async (importOriginal) => {
  const actualOs = await importOriginal<typeof osActual>();
  return {
    ...actualOs,
    homedir: vi.fn(() => '/mock/home/user'),
    platform: vi.fn(() => 'linux'),
  };
});
```

### 4.2 vi.hoisted - 測試前 Mock 創建

`vi.hoisted` 用於在模組導入之前創建 mock，解決 ES 模組的提升問題。

```typescript
// 在測試檔案頂部宣告（會被提升）
const mockInstallOrUpdateExtension: Mock<
  typeof ExtensionManager.prototype.installOrUpdateExtension
> = vi.hoisted(() => vi.fn());

const mockRequestConsent: Mock<typeof requestConsentNonInteractive> =
  vi.hoisted(() => vi.fn());

const mockStat: Mock<typeof fs.stat> = vi.hoisted(() => vi.fn());

const mockInferInstallMetadata: Mock<typeof inferInstallMetadata> =
  vi.hoisted(() => vi.fn());

// 然後在 vi.mock 中使用這些 mock
vi.mock('../../config/extensions/consent.js', () => ({
  requestConsentNonInteractive: mockRequestConsentNonInteractive,
}));

vi.mock('../../config/extension-manager.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('...')>();
  return {
    ...actual,
    ExtensionManager: vi.fn().mockImplementation(() => ({
      installOrUpdateExtension: mockInstallOrUpdateExtension,
      loadExtensions: vi.fn(),
    })),
    inferInstallMetadata: mockInferInstallMetadata,
  };
});

vi.mock('node:fs/promises', () => ({
  stat: mockStat,
  default: { stat: mockStat },
}));
```

**為什麼使用 vi.hoisted?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ES 模組執行順序問題                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  標準 JavaScript 執行順序:                                       │
│  1. 模組導入 (import)                                            │
│  2. vi.mock 執行                                                 │
│  3. 測試代碼執行                                                 │
│                                                                  │
│  問題: vi.mock 需要在導入之前執行，但 import 會被提升            │
│                                                                  │
│  vi.hoisted 解決方案:                                            │
│  1. vi.hoisted 內的代碼被提升到檔案頂部                          │
│  2. vi.mock 使用提升的 mock                                      │
│  3. 模組導入時已經有正確的 mock                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 vi.spyOn - 監視現有函數

```typescript
let debugLogSpy: MockInstance;
let debugErrorSpy: MockInstance;
let processSpy: MockInstance;

beforeEach(() => {
  // 監視日誌函數
  debugLogSpy = vi.spyOn(debugLogger, 'log');
  debugErrorSpy = vi.spyOn(debugLogger, 'error');

  // 監視 process.exit 並阻止實際退出
  processSpy = vi
    .spyOn(process, 'exit')
    .mockImplementation(() => undefined as never);
});

afterEach(() => {
  processSpy.mockRestore();
  debugLogSpy.mockRestore();
  debugErrorSpy.mockRestore();
});

it('should log error and exit on failure', async () => {
  mockInstallOrUpdateExtension.mockRejectedValue(
    new Error('Install extension failed'),
  );

  await handleInstall({ source: 'git@some-url' });

  expect(debugErrorSpy).toHaveBeenCalledWith('Install extension failed');
  expect(processSpy).toHaveBeenCalledWith(1);
});
```

### 4.4 MSW (Mock Service Worker) - HTTP 模擬

#### MSW 伺服器設置

**檔案路徑**: `/packages/core/src/mocks/msw.ts`

```typescript
import { setupServer } from 'msw/node';

export const server = setupServer();
```

#### 在測試中使用 MSW

**檔案路徑**: `/packages/cli/src/config/config.integration.test.ts`

```typescript
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

export const server = setupServer();

const CLEARCUT_URL = 'https://play.googleapis.com/log';

beforeAll(() => {
  server.listen({});  // 啟動攔截
});

afterEach(() => {
  server.resetHandlers();  // 重置處理器
});

afterAll(() => {
  server.close();  // 關閉伺服器
});

describe('Configuration Integration Tests', () => {
  beforeEach(() => {
    // 設置預設處理器
    server.resetHandlers(
      http.post(CLEARCUT_URL, () => HttpResponse.text())
    );
  });

  it('should handle API responses', async () => {
    // 可以為特定測試添加處理器
    server.use(
      http.get('/api/config', () => {
        return HttpResponse.json({ feature: true });
      })
    );

    // 測試代碼...
  });
});
```

### 4.5 自訂 Matcher

**檔案路徑**: `/packages/cli/src/test-utils/customMatchers.ts`

```typescript
import type { Assertion } from 'vitest';
import { expect } from 'vitest';
import type { TextBuffer } from '../ui/components/shared/text-buffer.js';

// 偵測無效字元的正則表達式
const invalidCharsRegex = /[\b\x1b]/;

/**
 * 自訂匹配器: 驗證 TextBuffer 只包含有效字元
 */
function toHaveOnlyValidCharacters(this: Assertion, buffer: TextBuffer) {
  const { isNot } = this as any;
  let pass = true;
  const invalidLines: Array<{ line: number; content: string }> = [];

  for (let i = 0; i < buffer.lines.length; i++) {
    const line = buffer.lines[i];
    // 檢查換行符
    if (line.includes('\n')) {
      pass = false;
      invalidLines.push({ line: i, content: line });
      break;
    }
    // 檢查無效字元
    if (invalidCharsRegex.test(line)) {
      pass = false;
      invalidLines.push({ line: i, content: line });
    }
  }

  return {
    pass,
    message: () =>
      `Expected buffer ${isNot ? 'not ' : ''}to have only valid characters, ` +
      `but found invalid characters in lines:\n${invalidLines
        .map((l) => `  [${l.line}]: "${l.content}"`)
        .join('\n')}`,
    actual: buffer.lines,
    expected: 'Lines with no line breaks, backspaces, or escape codes.',
  };
}

// 註冊自訂匹配器
expect.extend({
  toHaveOnlyValidCharacters,
} as any);

// 擴展 Vitest 的類型定義
declare module 'vitest' {
  interface Assertion<T> {
    toHaveOnlyValidCharacters(): T;
  }
  interface AsymmetricMatchersContaining {
    toHaveOnlyValidCharacters(): void;
  }
}
```

**使用範例**:

```typescript
it('should render without invalid characters', () => {
  const buffer = new TextBuffer();
  buffer.write('Hello World');

  expect(buffer).toHaveOnlyValidCharacters();
});
```

### 4.6 Mock 工具類

**檔案路徑**: `/packages/core/src/test-utils/mock-tool.ts`

```typescript
interface MockToolOptions {
  name: string;
  displayName?: string;
  description?: string;
  canUpdateOutput?: boolean;
  isOutputMarkdown?: boolean;
  shouldConfirmExecute?: (
    params: { [key: string]: unknown },
    signal: AbortSignal,
  ) => Promise<ToolCallConfirmationDetails | false>;
  execute?: (
    params: { [key: string]: unknown },
    signal?: AbortSignal,
    updateOutput?: (output: string) => void,
  ) => Promise<ToolResult>;
  params?: object;
}

/**
 * 高度可配置的 Mock 工具，用於測試
 */
export class MockTool extends BaseDeclarativeTool<
  { [key: string]: unknown },
  ToolResult
> {
  shouldConfirmExecute: (...) => Promise<...>;
  execute: (...) => Promise<ToolResult>;

  constructor(options: MockToolOptions) {
    super(
      options.name,
      options.displayName ?? options.name,
      options.description ?? options.name,
      Kind.Other,
      options.params,
      options.isOutputMarkdown ?? false,
      options.canUpdateOutput ?? false,
    );

    this.shouldConfirmExecute = options.shouldConfirmExecute
      ?? (() => Promise.resolve(false));

    this.execute = options.execute
      ?? (() => Promise.resolve({
        llmContent: `Tool ${this.name} executed successfully.`,
        returnDisplay: `Tool ${this.name} executed successfully.`,
      }));
  }
}

/**
 * 可修改的 Mock 工具，用於測試編輯功能
 */
export class MockModifiableTool
  extends BaseDeclarativeTool<Record<string, unknown>, ToolResult>
  implements ModifiableDeclarativeTool<Record<string, unknown>>
{
  executeFn: (params: Record<string, unknown>) => ToolResult | undefined;
  shouldConfirm = true;

  getModifyContext(signal: AbortSignal): ModifyContext<Record<string, unknown>> {
    return {
      getFilePath: () => 'test.txt',
      getCurrentContent: async () => 'old content',
      getProposedContent: async () => 'new content',
      createUpdatedParams: (oldContent, modifiedContent, params) =>
        ({ newContent: modifiedContent }),
    };
  }
}
```

---

## 5. 整合 vs 單元測試分離

### 5.1 測試分類架構

```
┌─────────────────────────────────────────────────────────────────────┐
│                        測試金字塔                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                         ▲                                            │
│                        /│\                                           │
│                       / │ \                                          │
│                      /  │  \     端對端整合測試                      │
│                     /   │   \    (integration-tests/)                │
│                    /────┼────\   24 個測試檔案                       │
│                   /     │     \  真實 CLI 調用                       │
│                  /──────┼──────\ 300s 超時                          │
│                 /       │       \                                    │
│                /        │        \  混合整合測試                     │
│               /─────────┼─────────\ (*.integration.test.ts)          │
│              /          │          \ ~10 個測試檔案                  │
│             /           │           \ 真實 FS + Mock API             │
│            /────────────┼────────────\                               │
│           /             │             \                              │
│          /              │              \    單元測試                 │
│         /───────────────┼───────────────\  (packages/*/src/**/)     │
│        /                │                \ ~507 個測試檔案           │
│       /                 │                 \ Mock 所有外部依賴        │
│      /──────────────────┼──────────────────\                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 單元測試特徵

**位置**: `packages/*/src/**/*.test.ts(x)`

**特徵**:
- 單一檔案聚焦一個模組/類別/函數
- Mock 所有外部依賴 (fs, network, 其他服務)
- 使用 mocked 檔案系統隔離執行
- 快速執行 (預設超時 5s，Core 30s)
- 無重試機制

**範例** (`CommandService.test.ts`):

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { CommandService } from './CommandService.js';
import type { ICommandLoader, SlashCommand } from './types.js';
import { debugLogger } from '@google/gemini-cli-core';

describe('CommandService', () => {
  // Mock 命令載入器
  class MockCommandLoader implements ICommandLoader {
    loadCommands = vi.fn(async (): Promise<SlashCommand[]> => [
      mockCommandA,
      mockCommandB,
    ]);
  }

  beforeEach(() => {
    vi.spyOn(debugLogger, 'debug').mockImplementation(() => {});
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('should load commands from a single loader', async () => {
    // Arrange
    const mockLoader = new MockCommandLoader();
    const signal = new AbortController().signal;

    // Act
    const service = await CommandService.create([mockLoader], signal);

    // Assert
    expect(mockLoader.loadCommands).toHaveBeenCalledTimes(1);
    expect(service.getCommands()).toHaveLength(2);
  });

  it('should merge commands from multiple loaders', async () => {
    // Arrange
    const loader1 = new MockCommandLoader();
    const loader2 = new MockCommandLoader();

    // Act
    const service = await CommandService.create([loader1, loader2], signal);

    // Assert
    expect(service.getCommands()).toHaveLength(4);
  });
});
```

### 5.3 混合整合測試特徵

**位置**: `packages/*/src/**/*.integration.test.ts`

**特徵**:
- 使用真實檔案系統（臨時目錄）
- Mock 網路請求（MSW）
- 測試配置載入和處理流程
- 中等執行時間

**範例** (`config.integration.test.ts`):

```typescript
import { beforeAll, afterAll, beforeEach, afterEach } from 'vitest';
import * as fs from 'node:fs';
import * as path from 'node:path';
import { tmpdir } from 'node:os';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

export const server = setupServer();

beforeAll(() => server.listen({}));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('Configuration Integration Tests', () => {
  let tempDir: string;

  beforeEach(() => {
    // 使用真實檔案系統
    tempDir = fs.mkdtempSync(path.join(tmpdir(), 'gemini-cli-test-'));

    // Mock HTTP 請求
    server.resetHandlers(
      http.post(CLEARCUT_URL, () => HttpResponse.text())
    );

    vi.stubEnv('GEMINI_API_KEY', 'test-api-key');
  });

  afterEach(() => {
    vi.unstubAllEnvs();
    if (fs.existsSync(tempDir)) {
      fs.rmSync(tempDir, { recursive: true });
    }
  });

  it.each([
    { fileFiltering: undefined, expected: true },
    { fileFiltering: { respectGitIgnore: false }, expected: false },
    { fileFiltering: { respectGitIgnore: true }, expected: true },
  ])('should load file filtering settings', async ({ fileFiltering, expected }) => {
    const config = new Config({
      sessionId: 'test-session',
      targetDir: tempDir,
      fileFiltering,
      // ...
    });

    expect(config.getFileFilteringRespectGitIgnore()).toBe(expected);
  });
});
```

### 5.4 端對端整合測試特徵

**位置**: `integration-tests/*.test.ts`

**特徵**:
- 透過 `TestRig` 完整 CLI 調用
- 真實檔案系統操作
- 模型回應 mocking (fake-responses.json goldens)
- 較長超時 (300s)
- 自動重試 (2 次)
- 完整工具執行流程
- 遙測驗證

**範例** (`file-system.test.ts`):

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { existsSync } from 'node:fs';
import * as path from 'node:path';
import { TestRig, printDebugInfo, validateModelOutput } from './test-helper.js';

describe('file-system', () => {
  let rig: TestRig;

  beforeEach(() => {
    rig = new TestRig();
  });

  afterEach(async () => await rig.cleanup());

  it('should be able to read a file', async () => {
    // Arrange - 設置測試環境
    await rig.setup('should be able to read a file', {
      settings: { tools: { core: ['read_file'] } },
    });
    rig.createFile('test.txt', 'hello world');

    // Act - 執行 CLI
    const result = await rig.run({
      args: `read the file test.txt and show me its contents`,
    });

    // Assert - 驗證工具調用
    const foundToolCall = await rig.waitForToolCall('read_file');

    // 調試資訊
    if (!foundToolCall || !result.includes('hello world')) {
      printDebugInfo(rig, result, {
        'Found tool call': foundToolCall,
        'Contains hello world': result.includes('hello world'),
      });
    }

    expect(foundToolCall, 'Expected to find a read_file tool call').toBeTruthy();
    validateModelOutput(result, 'hello world', 'File read test');
  });

  it('should perform a read-then-write sequence', async () => {
    await rig.setup('should perform a read-then-write sequence', {
      settings: { tools: { core: ['read_file', 'replace', 'write_file'] } },
    });
    const fileName = 'version.txt';
    rig.createFile(fileName, '1.0.0');

    const prompt = `Read the version from ${fileName} and write the next version 1.0.1 back to the file.`;
    const result = await rig.run({ args: prompt });

    await rig.waitForTelemetryReady();
    const toolLogs = rig.readToolLogs();

    const readCall = toolLogs.find(
      (log) => log.toolRequest.name === 'read_file',
    );
    const writeCall = toolLogs.find(
      (log) =>
        log.toolRequest.name === 'write_file' ||
        log.toolRequest.name === 'replace',
    );

    expect(readCall).toBeDefined();
    expect(writeCall).toBeDefined();

    const newFileContent = rig.readFile(fileName);
    expect(newFileContent).toBe('1.0.1');
  });
});
```

### 5.5 測試分類對比表

| 方面 | 單元測試 | 混合整合測試 | 端對端整合測試 |
|------|----------|--------------|----------------|
| **位置** | `packages/*/src/**/*.test.ts` | `*.integration.test.ts` | `integration-tests/*.test.ts` |
| **檔案系統** | Mocked (`vi.mock('fs')`) | 真實 (臨時目錄) | 真實 (`TestRig.testDir`) |
| **網路請求** | Mocked (`vi.mock`) | Mocked (MSW) | Mocked (`fake-responses.json`) |
| **CLI 調用** | N/A | N/A | 完整 (`spawn`/`pty`) |
| **超時** | 5s / 30s | 30s | 300s (5 分鐘) |
| **重試** | 否 | 否 | 是 (2 次) |
| **測試焦點** | 單一函數/類別 | 配置/功能流程 | 端對端使用者流程 |
| **執行速度** | 快 (毫秒級) | 中 (秒級) | 慢 (分鐘級) |
| **調試難度** | 低 | 中 | 高 |

---

## 6. 覆蓋率與 CI/CD 整合

### 6.1 覆蓋率配置

**覆蓋率提供者**: V8 (Node.js 內建)

**報告格式**:

| 格式 | 檔案 | 用途 |
|------|------|------|
| text | `full-text-summary.txt` | CLI 友好的文字摘要 |
| html | `index.html` | 互動式瀏覽器報告 |
| json | `coverage-summary.json` | 程式化讀取 |
| lcov | `lcov.info` | 標準覆蓋率格式 |
| cobertura | `cobertura-coverage.xml` | Jenkins/GitLab CI 格式 |

**配置範例**:

```typescript
coverage: {
  enabled: true,
  provider: 'v8',
  reportsDirectory: './coverage',
  include: ['src/**/*'],
  reporter: [
    ['text', { file: 'full-text-summary.txt' }],
    'html',
    'json',
    'lcov',
    'cobertura',
    ['json-summary', { outputFile: 'coverage-summary.json' }],
  ],
}
```

### 6.2 GitHub Actions CI 工作流程

**檔案路徑**: `/.github/workflows/ci.yml`

```yaml
name: 'Testing: CI'

on:
  push:
    branches: ['main', 'release/**']
  pull_request:
    branches: ['main', 'release/**']
  merge_group:
  workflow_dispatch:
    inputs:
      branch_ref:
        description: 'Branch to run on'
        default: 'main'

concurrency:
  group: '${{ github.workflow }}-${{ github.head_ref || github.ref }}'
  cancel-in-progress: |-
    ${{ github.ref != 'refs/heads/main' && !startsWith(github.ref, 'refs/heads/release/') }}

permissions:
  checks: 'write'
  contents: 'read'
  statuses: 'write'
```

### 6.3 CI Jobs 架構

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CI 工作流程架構                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    merge_queue_skipper                        │   │
│  │                    (檢查是否跳過 CI)                          │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                               │                                      │
│          ┌────────────────────┼────────────────────┐                │
│          │                    │                    │                 │
│          v                    v                    v                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐         │
│  │    lint      │     │  link_checker│     │   codeql     │         │
│  │              │     │              │     │              │         │
│  │  - ESLint    │     │  - Markdown  │     │  - JavaScript│         │
│  │  - Prettier  │     │    連結驗證   │     │    安全分析  │         │
│  │  - yamllint  │     │              │     │              │         │
│  │  - shellcheck│     │              │     │              │         │
│  │  - actionlint│     │              │     │              │         │
│  └──────────────┘     └──────────────┘     └──────────────┘         │
│          │                                                           │
│          v                                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      test_linux (矩陣)                        │   │
│  │                                                                │   │
│  │   Node 20.x ────┬─── Node 22.x ────┬─── Node 24.x             │   │
│  │       │         │        │          │        │                 │   │
│  │   ┌───v───┐ ┌───v───┐ ┌───v───┐                               │   │
│  │   │ Build │ │ Build │ │ Build │                               │   │
│  │   │ Test  │ │ Test  │ │ Test  │                               │   │
│  │   │ Bundle│ │ Bundle│ │ Bundle│                               │   │
│  │   └───────┘ └───────┘ └───────┘                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                               │                                      │
│          ┌────────────────────┼────────────────────┐                │
│          │                    │                    │                 │
│          v                    v                    v                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐         │
│  │  test_mac    │     │ test_windows │     │ bundle_size  │         │
│  │  (矩陣)      │     │  (Node 20.x) │     │  (PR only)   │         │
│  │              │     │              │     │              │         │
│  │ Node 20/22/24│     │ continue-on- │     │ 檢查打包大小 │         │
│  │              │     │ error: true  │     │ 變化         │         │
│  └──────────────┘     └──────────────┘     └──────────────┘         │
│                                                                      │
│                               │                                      │
│                               v                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                          ci                                   │   │
│  │                    (彙總所有結果)                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.4 測試 Jobs 詳細配置

#### test_linux Job

```yaml
test_linux:
  name: 'Test (Linux)'
  runs-on: 'gemini-cli-ubuntu-16-core'
  needs: ['lint', 'merge_queue_skipper']
  if: "${{needs.merge_queue_skipper.outputs.skip == 'false'}}"

  strategy:
    matrix:
      node-version: ['20.x', '22.x', '24.x']

  steps:
    - name: 'Checkout'
      uses: 'actions/checkout@v5'

    - name: 'Set up Node.js ${{ matrix.node-version }}'
      uses: 'actions/setup-node@v4'
      with:
        node-version: '${{ matrix.node-version }}'
        cache: 'npm'

    - name: 'Build project'
      run: 'npm run build'

    - name: 'Install dependencies for testing'
      run: 'npm ci'

    - name: 'Run tests and generate reports'
      env:
        NO_COLOR: true
      run: 'npm run test:ci'

    - name: 'Bundle'
      run: 'npm run bundle'

    - name: 'Smoke test bundle'
      run: 'node ./bundle/gemini.js --version'

    - name: 'Publish Test Report'
      uses: 'dorny/test-reporter@v2'
      with:
        name: 'Test Results (Node ${{ matrix.node-version }})'
        path: 'packages/*/junit.xml'
        reporter: 'java-junit'
        fail-on-error: 'false'

    - name: 'Upload coverage reports'
      uses: 'actions/upload-artifact@v4'
      with:
        name: 'coverage-reports-${{ matrix.node-version }}-${{ runner.os }}'
        path: 'packages/*/coverage'
```

#### test_windows Job (特殊處理)

```yaml
test_windows:
  name: 'Slow Test - Win'
  runs-on: 'gemini-cli-windows-16-core'
  continue-on-error: true  # Windows 測試失敗不阻止合併

  steps:
    - name: 'Configure Windows Defender exclusions'
      run: |
        Add-MpPreference -ExclusionPath $env:GITHUB_WORKSPACE -Force
        Add-MpPreference -ExclusionPath "$env:GITHUB_WORKSPACE\node_modules" -Force
        Add-MpPreference -ExclusionPath "$env:GITHUB_WORKSPACE\packages" -Force
        Add-MpPreference -ExclusionPath "$env:TEMP" -Force
      shell: 'pwsh'

    - name: 'Configure npm for Windows performance'
      run: |
        npm config set progress false
        npm config set audit false
        npm config set fund false
        npm config set loglevel error
        npm config set maxsockets 32
      shell: 'pwsh'

    - name: 'Run tests'
      env:
        NODE_OPTIONS: '--max-old-space-size=32768 --max-semi-space-size=256'
        UV_THREADPOOL_SIZE: '32'
      run: 'npm run test:ci -- --coverage.enabled=false'
      shell: 'pwsh'
```

### 6.5 Node 版本矩陣支援

| Node 版本 | 支援狀態 | 測試平台 |
|-----------|----------|----------|
| **20.x** | 完整支援 | Linux, macOS, Windows |
| **22.x** | 完整支援 | Linux, macOS, Windows |
| **24.x** | 完整支援 | Linux, macOS, Windows |

### 6.6 測試報告產出

| 產出類型 | 格式 | 用途 |
|----------|------|------|
| JUnit XML | `packages/*/junit.xml` | CI 測試報告視覺化 |
| 覆蓋率報告 | `packages/*/coverage/` | 程式碼覆蓋率分析 |
| 測試結果 | GitHub Checks | PR 狀態顯示 |

---

## 7. 測試模式與最佳實踐

### 7.1 AAA 模式 (Arrange-Act-Assert)

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

describe('Feature Name', () => {
  let mockSpy: MockInstance;

  beforeEach(() => {
    vi.clearAllMocks();
    mockSpy = vi.spyOn(obj, 'method').mockImplementation(...);
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('should do something specific', async () => {
    // ============ Arrange ============
    // 準備測試資料和環境
    const input = 'test-input';
    const expectedOutput = 'expected-output';

    // ============ Act ============
    // 執行被測試的功能
    const result = await functionUnderTest(input);

    // ============ Assert ============
    // 驗證結果
    expect(result).toBe(expectedOutput);
    expect(mockSpy).toHaveBeenCalledWith(input);
    expect(mockSpy).toHaveBeenCalledTimes(1);
  });
});
```

### 7.2 參數化測試 (it.each)

```typescript
describe('Approval Mode Integration Tests', () => {
  it.each([
    {
      description: 'should parse --approval-mode=auto_edit correctly',
      argv: ['node', 'script.js', '--approval-mode', 'auto_edit', '-p', 'test'],
      expected: { approvalMode: 'auto_edit', prompt: 'test', yolo: false },
    },
    {
      description: 'should parse --approval-mode=yolo correctly',
      argv: ['node', 'script.js', '--approval-mode', 'yolo', '-p', 'test'],
      expected: { approvalMode: 'yolo', prompt: 'test', yolo: false },
    },
    {
      description: 'should parse legacy --yolo flag correctly',
      argv: ['node', 'script.js', '--yolo', '-p', 'test'],
      expected: { yolo: true, approvalMode: undefined, prompt: 'test' },
    },
  ])('$description', async ({ argv, expected }) => {
    const originalArgv = process.argv;
    try {
      process.argv = argv;
      const parsedArgs = await parseArguments({} as Settings);

      expect(parsedArgs.approvalMode).toBe(expected.approvalMode);
      expect(parsedArgs.prompt).toBe(expected.prompt);
      expect(parsedArgs.yolo).toBe(expected.yolo);
    } finally {
      process.argv = originalArgv;
    }
  });
});
```

### 7.3 整合測試模式

#### 基本檔案系統測試

```typescript
it('should be able to read a file', async () => {
  // Setup
  await rig.setup('test name', {
    settings: { tools: { core: ['read_file'] } },
  });
  rig.createFile('test.txt', 'hello world');

  // Execute
  const result = await rig.run({
    args: `read the file test.txt`,
  });

  // Verify tool call
  const foundToolCall = await rig.waitForToolCall('read_file');
  expect(foundToolCall).toBeTruthy();

  // Verify output
  validateModelOutput(result, 'hello world');
});
```

#### 互動式測試

```typescript
it('should perform a read-then-write sequence', async () => {
  const fileName = 'version.txt';
  await rig.setup('interactive-test', { settings: {...} });
  rig.createFile(fileName, '1.0.0');

  // 啟動互動式 CLI
  const run = await rig.runInteractive();

  // 步驟 1: 輸入讀取命令
  await run.type(`Read the version from ${fileName}`);
  await run.sendKeys('\r');

  // 等待工具調用
  const readCall = await rig.waitForToolCall('read_file', 30000);
  expect(readCall).toBe(true);

  // 步驟 2: 輸入寫入命令
  await run.type(`now change the version to 1.0.1 in the file`);
  await run.sendKeys('\r');

  // 驗證寫入成功
  await rig.expectToolCallSuccess(
    ['write_file', 'replace'],
    30000,
    (args) => args.includes('1.0.1') && args.includes(fileName),
  );

  // 等待遙測同步
  await rig.waitForTelemetryReady();
});
```

#### Hook 系統測試

```typescript
it('should block tool execution when hook returns block decision', async () => {
  await rig.setup('hook test', {
    fakeResponsesPath: join(import.meta.dirname, 'fake-responses.json'),
    settings: {
      tools: { enableHooks: true },
      hooks: {
        BeforeTool: [{
          matcher: 'write_file',
          sequential: true,
          hooks: [{
            type: 'command',
            command: `node -e "console.log(JSON.stringify({
              decision: 'block',
              reason: 'File writing blocked by security policy'
            }))"`,
            timeout: 5000,
          }],
        }],
      },
    },
  });

  const result = await rig.run({
    args: 'Create a file called test.txt with content "Hello World"',
  });

  // 驗證工具被阻止
  const toolLogs = rig.readToolLogs();
  const writeFileCalls = toolLogs.filter(
    (t) => t.toolRequest.name === 'write_file' && t.toolRequest.success,
  );
  expect(writeFileCalls).toHaveLength(0);

  // 驗證阻止原因
  expect(result).toContain('File writing blocked by security policy');

  // 驗證 Hook 遙測
  const hookTelemetryFound = await rig.waitForTelemetryEvent('hook_call');
  expect(hookTelemetryFound).toBeTruthy();
});
```

### 7.4 調試模式

#### 使用 printDebugInfo

```typescript
it('should handle complex scenario', async () => {
  await rig.setup('complex test', { ... });

  const result = await rig.run({ args: '...' });
  const foundToolCall = await rig.waitForToolCall('expected_tool');

  // 當測試可能失敗時打印調試資訊
  if (!foundToolCall || !result.includes('expected text')) {
    printDebugInfo(rig, result, {
      'Found tool call': foundToolCall,
      'Contains expected text': result.includes('expected text'),
      'Result length': result.length,
    });
  }

  expect(foundToolCall).toBeTruthy();
});
```

#### 保留測試輸出

```bash
# 保留測試產物以供檢查
KEEP_OUTPUT=true npm run test:e2e

# 詳細日誌模式
VERBOSE=true npm run test:e2e

# 同時使用
KEEP_OUTPUT=true VERBOSE=true npm run test:e2e
```

#### 讀取工具日誌

```typescript
// 讀取所有工具調用
const toolLogs = rig.readToolLogs();
console.log('All tools:', toolLogs.map(t => t.toolRequest.name));

// 讀取 Hook 日誌
const hookLogs = rig.readHookLogs();
console.log('Hook events:', hookLogs.map(h => h.hookCall.hook_event_name));

// 讀取 API 請求
const apiRequests = rig.readAllApiRequest();
console.log('API calls:', apiRequests.length);
```

### 7.5 條件跳過測試

```typescript
// 根據平台跳過
it.skipIf(process.platform === 'win32')(
  'should work on Unix-like systems',
  async () => { ... }
);

// 根據環境變數跳過
it.skipIf(process.env.CI === 'true')(
  'should only run locally',
  async () => { ... }
);

// 手動跳過（待實作）
it.skip('should replace multiple instances', async () => { ... });
```

---

## 8. 整合測試輔助框架

### 8.1 TestRig 類別架構

**檔案路徑**: `/integration-tests/test-helper.ts` (1118 行)

```typescript
export class TestRig {
  testDir: string | null = null;
  testName?: string;
  _lastRunStdout?: string;
  fakeResponsesPath?: string;
  originalFakeResponsesPath?: string;
  private _interactiveRuns: InteractiveRun[] = [];

  // ==================== 設置方法 ====================

  /**
   * 設置測試環境
   */
  setup(
    testName: string,
    options: {
      settings?: Record<string, unknown>;
      fakeResponsesPath?: string;
    } = {},
  ) {
    this.testName = testName;
    const sanitizedName = sanitizeTestName(testName);
    const testFileDir = env['INTEGRATION_TEST_FILE_DIR'] || join(os.tmpdir(), 'gemini-cli-tests');
    this.testDir = join(testFileDir, sanitizedName);
    mkdirSync(this.testDir, { recursive: true });

    // 複製 fake responses 檔案
    if (options.fakeResponsesPath) {
      this.fakeResponsesPath = join(this.testDir, 'fake-responses.json');
      this.originalFakeResponsesPath = options.fakeResponsesPath;
      if (process.env['REGENERATE_MODEL_GOLDENS'] !== 'true') {
        fs.copyFileSync(options.fakeResponsesPath, this.fakeResponsesPath);
      }
    }

    // 創建設置檔案
    const geminiDir = join(this.testDir, GEMINI_DIR);
    mkdirSync(geminiDir, { recursive: true });

    const settings = {
      general: { disableAutoUpdate: true, previewFeatures: false },
      telemetry: { enabled: true, target: 'local', outfile: telemetryPath },
      security: { auth: { selectedType: 'gemini-api-key' } },
      ui: { useAlternateBuffer: true },
      model: DEFAULT_GEMINI_MODEL,
      sandbox: env['GEMINI_SANDBOX'] !== 'false' ? env['GEMINI_SANDBOX'] : false,
      ide: { enabled: false, hasSeenNudge: true },
      ...options.settings,
    };
    writeFileSync(join(geminiDir, 'settings.json'), JSON.stringify(settings, null, 2));
  }

  /**
   * 創建測試檔案
   */
  createFile(fileName: string, content: string) {
    const filePath = join(this.testDir!, fileName);
    writeFileSync(filePath, content);
    return filePath;
  }

  /**
   * 創建目錄
   */
  mkdir(dir: string) {
    mkdirSync(join(this.testDir!, dir), { recursive: true });
  }

  /**
   * 同步檔案系統
   */
  sync() {
    execSync('sync', { cwd: this.testDir! });
  }

  // ==================== 執行方法 ====================

  /**
   * 執行 CLI（非互動模式）
   */
  run(options: {
    args?: string | string[];
    stdin?: string;
    stdinDoesNotEnd?: boolean;
    yolo?: boolean;
  }): Promise<string> {
    const yolo = options.yolo !== false;
    const { command, initialArgs } = this._getCommandAndArgs(yolo ? ['--yolo'] : []);

    const child = spawn(command, [...initialArgs, ...(options.args || [])], {
      cwd: this.testDir!,
      stdio: 'pipe',
      env: env,
    });

    // 處理 stdin、stdout、stderr...
    return new Promise((resolve, reject) => {
      child.on('close', (code) => {
        if (code === 0) resolve(stdout);
        else reject(new Error(`Process exited with code ${code}:\n${stderr}`));
      });
    });
  }

  /**
   * 執行 CLI 命令
   */
  runCommand(args: string[], options: { stdin?: string } = {}): Promise<string> {
    // 類似 run() 但不帶 --yolo
  }

  /**
   * 啟動互動式 CLI
   */
  async runInteractive(options?: {
    args?: string | string[];
    yolo?: boolean;
  }): Promise<InteractiveRun> {
    const yolo = options?.yolo !== false;
    const { command, initialArgs } = this._getCommandAndArgs(yolo ? ['--yolo'] : []);

    const ptyProcess = pty.spawn(executable, commandArgs, {
      name: 'xterm-color',
      cols: 80,
      rows: 80,
      cwd: this.testDir!,
      env: { ... },
    });

    const run = new InteractiveRun(ptyProcess);
    this._interactiveRuns.push(run);

    // 等待 CLI 就緒
    await run.expectText('Type your message or @path/to/file', 30000);
    return run;
  }

  // ==================== 結果檢查 ====================

  /**
   * 讀取檔案內容
   */
  readFile(fileName: string): string {
    const filePath = join(this.testDir!, fileName);
    return readFileSync(filePath, 'utf-8');
  }

  /**
   * 讀取工具調用日誌
   */
  readToolLogs(): ToolLog[] {
    const parsedLogs = this._readAndParseTelemetryLog();
    // 篩選 tool_call 事件...
  }

  /**
   * 讀取 Hook 日誌
   */
  readHookLogs(): HookLog[] {
    const parsedLogs = this._readAndParseTelemetryLog();
    // 篩選 hook_call 事件...
  }

  /**
   * 讀取 API 請求日誌
   */
  readAllApiRequest(): ParsedLog[] {
    const logs = this._readAndParseTelemetryLog();
    return logs.filter(log =>
      log.attributes?.['event.name'] === 'gemini_cli.api_request'
    );
  }

  // ==================== 非同步等待 ====================

  /**
   * 等待遙測就緒
   */
  async waitForTelemetryReady() {
    const logFilePath = join(this.testDir!, 'telemetry.log');
    await poll(
      () => {
        if (!fs.existsSync(logFilePath)) return false;
        const content = readFileSync(logFilePath, 'utf-8');
        return content.includes('"scopeMetrics"');
      },
      2000, 100
    );
  }

  /**
   * 等待特定工具調用
   */
  async waitForToolCall(
    toolName: string,
    timeout?: number,
    matchArgs?: (args: string) => boolean,
  ): Promise<boolean> {
    if (!timeout) timeout = getDefaultTimeout();
    await this.waitForTelemetryReady();

    return poll(
      () => {
        const toolLogs = this.readToolLogs();
        return toolLogs.some(
          (log) =>
            log.toolRequest.name === toolName &&
            (matchArgs?.call(this, log.toolRequest.args) ?? true),
        );
      },
      timeout, 100
    );
  }

  /**
   * 等待任一工具調用
   */
  async waitForAnyToolCall(toolNames: string[], timeout?: number): Promise<boolean> {
    // 類似 waitForToolCall 但接受多個工具名稱
  }

  /**
   * 等待遙測事件
   */
  async waitForTelemetryEvent(eventName: string, timeout?: number): Promise<boolean> {
    await this.waitForTelemetryReady();
    return poll(
      () => {
        const logs = this._readAndParseTelemetryLog();
        return logs.some(
          (logData) => logData.attributes?.['event.name'] === `gemini_cli.${eventName}`
        );
      },
      timeout ?? getDefaultTimeout(), 100
    );
  }

  /**
   * 期望工具調用成功
   */
  async expectToolCallSuccess(
    toolNames: string[],
    timeout?: number,
    matchArgs?: (args: string) => boolean,
  ) {
    // 等待並驗證工具成功執行
  }

  /**
   * 等待指標
   */
  async waitForMetric(metricName: string, timeout?: number): Promise<boolean> {
    // 等待特定指標出現在遙測日誌中
  }

  // ==================== 清理 ====================

  /**
   * 清理測試環境
   */
  async cleanup() {
    // 終止所有互動式執行
    for (const run of this._interactiveRuns) {
      try { await run.kill(); } catch (error) { }
    }
    this._interactiveRuns = [];

    // 如果在錄製模式，複製 fake responses 回原位
    if (process.env['REGENERATE_MODEL_GOLDENS'] === 'true' && this.fakeResponsesPath) {
      fs.copyFileSync(this.fakeResponsesPath, this.originalFakeResponsesPath!);
    }

    // 清理測試目錄
    if (this.testDir && !env['KEEP_OUTPUT']) {
      try {
        fs.rmSync(this.testDir, { recursive: true, force: true });
      } catch (error) { }
    }
  }
}
```

### 8.2 InteractiveRun 類別

```typescript
export class InteractiveRun {
  ptyProcess: pty.IPty;
  public output = '';

  constructor(ptyProcess: pty.IPty) {
    this.ptyProcess = ptyProcess;
    ptyProcess.onData((data) => {
      this.output += data;
      if (env['KEEP_OUTPUT'] === 'true' || env['VERBOSE'] === 'true') {
        process.stdout.write(data);
      }
    });
  }

  /**
   * 等待特定文字出現
   */
  async expectText(text: string, timeout?: number) {
    if (!timeout) timeout = getDefaultTimeout();
    await poll(
      () => stripAnsi(this.output).toLowerCase().includes(text.toLowerCase()),
      timeout, 200
    );
    expect(stripAnsi(this.output).toLowerCase()).toContain(text.toLowerCase());
  }

  /**
   * 逐字輸入（等待回顯）
   */
  async type(text: string) {
    let typedSoFar = '';
    for (const char of text) {
      this.ptyProcess.write(char);
      typedSoFar += char;

      const found = await poll(
        () => stripAnsi(this.output).includes(typedSoFar),
        5000, 10
      );

      if (!found) {
        throw new Error(
          `Timed out waiting for typed text: "${typedSoFar}".\nOutput:\n${stripAnsi(this.output)}`
        );
      }
    }
  }

  /**
   * 一次發送整個字串
   */
  async sendText(text: string) {
    this.ptyProcess.write(text);
    await new Promise((resolve) => setTimeout(resolve, 5));
  }

  /**
   * 逐字發送（不等待回顯）
   */
  async sendKeys(text: string) {
    const delay = 5;
    for (const char of text) {
      this.ptyProcess.write(char);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  /**
   * 終止進程
   */
  async kill() {
    this.ptyProcess.kill();
  }

  /**
   * 等待進程退出
   */
  expectExit(): Promise<number> {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(
        () => reject(new Error('Test timed out: process did not exit within a minute.')),
        60000
      );
      this.ptyProcess.onExit(({ exitCode }) => {
        clearTimeout(timer);
        resolve(exitCode);
      });
    });
  }
}
```

### 8.3 輔助函數

```typescript
/**
 * 輪詢等待條件成立
 */
export async function poll(
  predicate: () => boolean,
  timeout: number,
  interval: number,
): Promise<boolean> {
  const startTime = Date.now();
  let attempts = 0;
  while (Date.now() - startTime < timeout) {
    attempts++;
    const result = predicate();
    if (env['VERBOSE'] === 'true' && attempts % 5 === 0) {
      console.log(`Poll attempt ${attempts}: ${result ? 'success' : 'waiting...'}`);
    }
    if (result) return true;
    await new Promise((resolve) => setTimeout(resolve, interval));
  }
  return false;
}

/**
 * 創建工具調用錯誤訊息
 */
export function createToolCallErrorMessage(
  expectedTools: string | string[],
  foundTools: string[],
  result: string,
): string {
  const expectedStr = Array.isArray(expectedTools)
    ? expectedTools.join(' or ')
    : expectedTools;
  return (
    `Expected to find ${expectedStr} tool call(s). ` +
    `Found: ${foundTools.length > 0 ? foundTools.join(', ') : 'none'}. ` +
    `Output preview: ${result?.substring(0, 200) || 'no output'}...`
  );
}

/**
 * 打印調試資訊
 */
export function printDebugInfo(
  rig: TestRig,
  result: string,
  context: Record<string, unknown> = {},
) {
  console.error('Test failed - Debug info:');
  console.error('Result length:', result.length);
  console.error('Result (first 500 chars):', result.substring(0, 500));
  console.error('Result (last 500 chars):', result.substring(result.length - 500));

  Object.entries(context).forEach(([key, value]) => {
    console.error(`${key}:`, value);
  });

  const allTools = rig.readToolLogs();
  console.error('All tool calls found:', allTools.map((t) => t.toolRequest.name));
  return allTools;
}

/**
 * 驗證模型輸出
 */
export function validateModelOutput(
  result: string,
  expectedContent: string | (string | RegExp)[] | null = null,
  testName = '',
): boolean {
  if (!result || result.trim().length === 0) {
    throw new Error('Expected LLM to return some output');
  }

  if (expectedContent) {
    const contents = Array.isArray(expectedContent) ? expectedContent : [expectedContent];
    const missingContent = contents.filter((content) => {
      if (typeof content === 'string') {
        return !result.toLowerCase().includes(content.toLowerCase());
      } else if (content instanceof RegExp) {
        return !content.test(result);
      }
      return false;
    });

    if (missingContent.length > 0) {
      console.warn(
        `Warning: LLM did not include expected content: ${missingContent.join(', ')}.`,
        'The tool was called successfully, which is the main requirement.'
      );
      return false;
    }
  }

  return true;
}
```

---

## 9. 沙盒模式

### 9.1 沙盒模式概述

整合測試支援三種沙盒模式，透過 `GEMINI_SANDBOX` 環境變數控制：

| 模式 | 環境變數值 | 描述 | 使用場景 |
|------|------------|------|----------|
| **none** | `false` | 直接在主機執行 | 本地開發、快速測試 |
| **docker** | `docker` | Docker 容器沙盒 | CI 標準測試 |
| **podman** | `podman` | Podman 容器沙盒 | 無 root CI 環境 |

### 9.2 沙盒模式執行命令

```bash
# 無沙盒模式
GEMINI_SANDBOX=false vitest run --root ./integration-tests

# Docker 沙盒模式
GEMINI_SANDBOX=docker npm run build:sandbox && \
GEMINI_SANDBOX=docker vitest run --root ./integration-tests

# Podman 沙盒模式
GEMINI_SANDBOX=podman vitest run --root ./integration-tests

# 執行所有沙盒模式
npm run test:integration:all
```

### 9.3 沙盒相關配置

```typescript
// TestRig.setup() 中的沙盒配置
const settings = {
  // ...
  sandbox: env['GEMINI_SANDBOX'] !== 'false'
    ? env['GEMINI_SANDBOX']
    : false,
  // ...
};
```

### 9.4 Podman 特殊處理

Podman 模式需要特殊的遙測解析邏輯，因為輸出格式可能不同：

```typescript
readToolLogs() {
  // For Podman, first check if telemetry file exists and has content
  // If not, fall back to parsing from stdout
  if (env['GEMINI_SANDBOX'] === 'podman') {
    const logFilePath = join(this.testDir!, 'telemetry.log');

    if (fs.existsSync(logFilePath)) {
      try {
        const content = readFileSync(logFilePath, 'utf-8');
        if (content && content.includes('"event.name"')) {
          // File has content, use normal file parsing
        } else if (this._lastRunStdout) {
          // File empty, parse from stdout
          return this._parseToolLogsFromStdout(this._lastRunStdout);
        }
      } catch {
        if (this._lastRunStdout) {
          return this._parseToolLogsFromStdout(this._lastRunStdout);
        }
      }
    } else if (this._lastRunStdout) {
      return this._parseToolLogsFromStdout(this._lastRunStdout);
    }
  }
  // ... 正常解析邏輯
}
```

---

## 10. 環境變數和超時調整

### 10.1 環境變數總覽

| 環境變數 | 預設值 | 描述 | 使用位置 |
|----------|--------|------|----------|
| `CI` | - | CI 環境標識 | 超時計算、日誌行為 |
| `NO_COLOR` | - | 停用顏色輸出 | test-setup, globalSetup |
| `GEMINI_SANDBOX` | `false` | 沙盒模式 | TestRig, 超時計算 |
| `GEMINI_CONFIG_DIR` | - | 配置目錄 | globalSetup |
| `GEMINI_CLI_INTEGRATION_TEST` | `true` | 整合測試標識 | globalSetup |
| `GEMINI_FORCE_FILE_STORAGE` | `true` | 強制檔案儲存 | globalSetup (避免 keychain) |
| `INTEGRATION_TEST_FILE_DIR` | - | 測試檔案目錄 | TestRig |
| `TELEMETRY_LOG_FILE` | - | 遙測日誌路徑 | globalSetup |
| `KEEP_OUTPUT` | `false` | 保留測試產物 | cleanup, 日誌輸出 |
| `VERBOSE` | `false` | 詳細日誌 | 輪詢, 日誌輸出 |
| `REGENERATE_MODEL_GOLDENS` | `false` | 重新錄製模型回應 | TestRig |

### 10.2 超時調整邏輯

```typescript
/**
 * 根據環境動態計算超時時間
 */
function getDefaultTimeout() {
  if (env['CI']) return 60000;           // CI 環境: 1 分鐘
  if (env['GEMINI_SANDBOX']) return 30000; // 沙盒環境: 30 秒
  return 15000;                           // 本地開發: 15 秒
}
```

### 10.3 超時配置摘要

| 測試類型 | Vitest 配置超時 | 動態超時 (getDefaultTimeout) |
|----------|-----------------|------------------------------|
| CLI 單元測試 | 5s (預設) | N/A |
| Core 單元測試 | 30s | N/A |
| 整合測試 (Vitest) | 300s | - |
| 整合測試 (本地) | - | 15s |
| 整合測試 (沙盒) | - | 30s |
| 整合測試 (CI) | - | 60s |

### 10.4 調試環境變數使用

```bash
# 保留所有測試輸出以供調試
KEEP_OUTPUT=true npm run test:e2e

# 啟用詳細日誌
VERBOSE=true npm run test:e2e

# 組合使用（最大調試資訊）
KEEP_OUTPUT=true VERBOSE=true npm run test:e2e

# 重新錄製模型回應 (golden files)
REGENERATE_MODEL_GOLDENS=true npm run test:integration:sandbox:none
```

---

## 11. 總結統計

### 11.1 測試檔案統計

| 指標 | 數量 |
|------|------|
| **總測試檔案數** | 541 |
| **單元測試套件** | 4 (cli, core, a2a-server, test-utils) |
| **整合測試套件** | 1 (integration-tests/) |
| **Vitest 配置檔案** | 6 |

### 11.2 配置參數統計

| 指標 | 數值 |
|------|------|
| **超時範圍** | 5s - 300s |
| **執行緒池** | 8-16 執行緒 |
| **整合測試重試** | 2 次 |
| **覆蓋率報告器** | 6 種格式 |
| **沙盒模式** | 3 種 (none, docker, podman) |

### 11.3 CI/CD 統計

| 指標 | 數量 |
|------|------|
| **CI Node 版本** | 3 (20.x, 22.x, 24.x) |
| **測試平台** | 3 (Linux, macOS, Windows) |
| **Lint 檢查類型** | 5 (ESLint, Prettier, yamllint, shellcheck, actionlint) |

### 11.4 工具和輔助函數統計

| 類別 | 數量 |
|------|------|
| **Mock 工具類** | 2 (MockTool, MockModifiableTool) |
| **測試輔助類** | 2 (TestRig, InteractiveRun) |
| **輔助函數** | ~10 (poll, printDebugInfo, validateModelOutput, etc.) |
| **自訂 Matcher** | 1 (toHaveOnlyValidCharacters) |
| **React 測試工具** | 4 (render, renderWithProviders, renderHook, simulateClick) |

---

## 附錄 A: 常用測試命令速查表

```bash
# 執行所有測試
npm test

# CI 模式測試（含覆蓋率）
npm run test:ci

# 端對端測試（詳細輸出）
npm run test:e2e

# 整合測試（無沙盒）
npm run test:integration:sandbox:none

# 整合測試（Docker 沙盒）
npm run test:integration:sandbox:docker

# 整合測試（Podman 沙盒）
npm run test:integration:sandbox:podman

# 執行所有沙盒模式
npm run test:integration:all

# 穩定性測試（deflake）
npm run deflake:test:integration:sandbox:none

# 腳本測試
npm run test:scripts

# 特定套件測試
npm run test --workspace @google/gemini-cli-cli
npm run test --workspace @google/gemini-cli-core
```

## 附錄 B: 測試檔案範本

### 單元測試範本

```typescript
/**
 * @license
 * Copyright 2025 Google LLC
 * SPDX-License-Identifier: Apache-2.0
 */

import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

// vi.hoisted mock 宣告
const mockDependency = vi.hoisted(() => vi.fn());

// vi.mock 模組替換
vi.mock('./dependency.js', () => ({
  dependency: mockDependency,
}));

describe('MyFeature', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('should do something', () => {
    // Arrange
    mockDependency.mockReturnValue('mocked value');

    // Act
    const result = functionUnderTest();

    // Assert
    expect(result).toBe('expected');
    expect(mockDependency).toHaveBeenCalledWith('arg');
  });

  it.each([
    { input: 'a', expected: 'A' },
    { input: 'b', expected: 'B' },
  ])('should handle $input', ({ input, expected }) => {
    expect(transform(input)).toBe(expected);
  });
});
```

### 整合測試範本

```typescript
/**
 * @license
 * Copyright 2025 Google LLC
 * SPDX-License-Identifier: Apache-2.0
 */

import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { TestRig, printDebugInfo, validateModelOutput } from './test-helper.js';
import { join } from 'node:path';

describe('MyFeature Integration', () => {
  let rig: TestRig;

  beforeEach(() => {
    rig = new TestRig();
  });

  afterEach(async () => {
    await rig.cleanup();
  });

  it('should perform expected action', async () => {
    // Setup
    await rig.setup('test name', {
      fakeResponsesPath: join(import.meta.dirname, 'fake-responses.json'),
      settings: {
        tools: { core: ['read_file', 'write_file'] },
      },
    });
    rig.createFile('test.txt', 'initial content');

    // Execute
    const result = await rig.run({
      args: 'Do something with test.txt',
    });

    // Verify tool calls
    const foundToolCall = await rig.waitForToolCall('expected_tool');
    if (!foundToolCall) {
      printDebugInfo(rig, result);
    }
    expect(foundToolCall).toBeTruthy();

    // Verify output
    validateModelOutput(result, 'expected text');

    // Verify file changes
    const fileContent = rig.readFile('test.txt');
    expect(fileContent).toContain('expected content');
  });
});
```
