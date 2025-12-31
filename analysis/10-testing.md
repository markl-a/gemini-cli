# 測試架構深度分析報告

## 1. 測試框架設置 (Vitest 配置)

**框架:** Vitest 3.2.4 (現代 Vite 原生測試框架)

### 1.1 主要 Vitest 配置

**A. CLI 套件配置** (`/packages/cli/vitest.config.ts`, 58 行)
```typescript
- environment: 'node'
- globals: true (describe, it, expect 全域可用)
- include: ['**/*.{test,spec}.{js,ts,jsx,tsx}', 'config.test.ts']
- exclude: ['**/node_modules/**', '**/dist/**']
- reporters: ['default', 'junit']
- outputFile: { junit: 'junit.xml' }
- setupFiles: ['./test-setup.ts']
- poolOptions: 8-16 執行緒池用於並行執行
- coverage provider: v8
```

**B. Core 套件配置** (`/packages/core/vitest.config.ts`, 39 行)
```typescript
- reporters: ['default', 'junit']
- timeout: 30000ms
- silent: true
- setupFiles: ['./test-setup.ts']
- coverage: 多種報告器 (text, html, json, lcov, cobertura)
```

**C. 整合測試配置** (`/integration-tests/vitest.config.ts`, 24 行)
```typescript
- testTimeout: 300000ms (5 分鐘 - 整合測試較長)
- globalSetup: './globalSetup.ts'
- fileParallelism: true
- retry: 2 (自動重試失敗測試)
- poolOptions: 8-16 執行緒
```

### 1.2 測試腳本 (package.json)

```json
{
  "test": "npm run test --workspaces --if-present",
  "test:ci": "npm run test:ci --workspaces --if-present && npm run test:scripts",
  "test:scripts": "vitest run --config ./scripts/tests/vitest.config.ts",
  "test:e2e": "cross-env VERBOSE=true KEEP_OUTPUT=true npm run test:integration:sandbox:none",
  "test:integration:all": "npm run test:integration:sandbox:none && npm run test:integration:sandbox:docker && npm run test:integration:sandbox:podman",
  "test:integration:sandbox:none": "cross-env GEMINI_SANDBOX=false vitest run --root ./integration-tests",
  "test:integration:sandbox:docker": "cross-env GEMINI_SANDBOX=docker npm run build:sandbox && vitest run --root ./integration-tests"
}
```

---

## 2. 測試檔案組織與模式

**總測試檔案:** 541 個測試檔案

### 目錄結構

```
gemini-cli/
├── packages/
│   ├── cli/
│   │   └── src/
│   │       ├── config/
│   │       │   ├── config.test.ts (單元測試)
│   │       │   ├── config.integration.test.ts (整合測試)
│   │       │   └── settings.test.ts
│   │       ├── commands/
│   │       │   ├── extensions.test.tsx
│   │       │   └── extensions/
│   │       │       ├── install.test.ts
│   │       │       └── uninstall.test.ts
│   │       ├── services/
│   │       │   ├── CommandService.test.ts
│   │       │   └── FileCommandLoader.test.ts
│   │       ├── ui/
│   │       │   ├── App.test.tsx
│   │       │   └── commands/
│   │       │       ├── authCommand.test.ts
│   │       │       └── *.test.ts (40+ 命令測試)
│   │       └── test-utils/
│   ├── core/
│   │   └── src/
│   │       └── test-utils/
│   │           ├── mock-tool.ts
│   │           └── mock-message-bus.ts
│   └── test-utils/
│       └── src/
│           └── file-system-test-helpers.ts
└── integration-tests/
    ├── globalSetup.ts
    ├── test-helper.ts (1118 行 - 核心測試框架)
    ├── file-system.test.ts
    ├── hooks-system.test.ts
    └── 20+ 更多整合測試
```

**命名慣例:**
- 單元測試: `*.test.ts` 或 `*.test.tsx`
- 整合測試: `*.test.ts` 在 `/integration-tests`
- 混合測試: `.integration.test.ts` 後綴

---

## 3. 測試工具與輔助函數

### A. 全域測試設置檔案

**CLI test-setup.ts (65 行)**
```typescript
- IS_REACT_ACT_ENVIRONMENT = true (React 測試模式)
- NO_COLOR 環境清理
- 自訂 matchers 導入
- beforeEach hook: 監視 console.error 偵測 act() 警告
- afterEach hook: 對 "was not wrapped in act(...)" 警告失敗測試
```

**整合測試 globalSetup.ts (84 行)**
```typescript
// 設置階段:
- 創建唯一測試執行目錄: .integration-tests/{timestamp}/
- 設置 HOME 環境變數到測試目錄
- 設置 GEMINI_CONFIG_DIR
- 下載 ripgrep 二進位檔
- 清理舊測試執行 (保留最新 5 個)
- 設置測試特定環境變數

// 清理階段:
- 移除測試執行目錄 (除非 KEEP_OUTPUT=true)
```

### B. 測試工具套件

**file-system-test-helpers.ts (99 行)**

```typescript
export type FileSystemStructure = {
  [name: string]:
    | string                    // 檔案內容
    | FileSystemStructure       // 子目錄
    | Array<string | FileSystemStructure>  // 帶檔案的目錄
};

// 關鍵函數:
- createTmpDir(structure): 使用檔案結構創建臨時目錄
- cleanupTmpDir(dir): 移除臨時目錄
- async create(dir, structure): 遞迴創建檔案/目錄
```

**使用範例:**
```typescript
const structure = {
  'file1.txt': 'Hello, world!',
  'src': {
    'main.js': '// Main file',
    'utils.ts': '// Utils'
  },
  'data': ['users.csv', 'products.json']
};
const tmpDir = await createTmpDir(structure);
```

### C. React 元件測試工具

**render.tsx (200+ 行)**

```typescript
export const render = (
  tree: React.ReactElement,
  terminalWidth?: number
): ReturnType<typeof inkRender>

- 使用 act() 包裝所有渲染
- 允許終端機寬度配置
- simulateClick(stdin, col, row, button): 滑鼠事件模擬
- mockConfig: Config 物件的預設 mock
- createMockSettings(overrides): 帶深度合併的 Settings 工廠
```

**自訂 waitFor 實現 (async.ts, 35 行)**

```typescript
export async function waitFor(
  assertion: () => void,
  { timeout = 1000, interval = 50 } = {}
): Promise<void>

- 使用 React.act() 包裝斷言
- 可配置的超時/間隔輪詢
- 修復 vitest 的 waitFor 不包裝 act() 的問題
```

---

## 4. Mock 策略

### A. 模組 Mocking 模式 (vi.mock)

**完整模組替換:**
```typescript
vi.mock('./trustedFolders.js', () => ({
  isWorkspaceTrusted: vi.fn(() => ({ isTrusted: true, source: 'file' })),
}));

vi.mock('fs', async (importOriginal) => {
  const actualFs = await importOriginal<typeof import('fs')>();
  return {
    ...actualFs,
    mkdirSync: vi.fn(),
    writeFileSync: vi.fn(),
    existsSync: vi.fn((p) => mockPaths.has(p.toString())),
  };
});
```

**部分模組替換:**
```typescript
vi.mock('os', async (importOriginal) => {
  const actualOs = await importOriginal<typeof osActual>();
  return {
    ...actualOs,  // 保留所有原始函數
    homedir: vi.fn(() => '/mock/home/user'),  // 只覆蓋這個
    platform: vi.fn(() => 'linux'),
  };
});
```

### B. vi.hoisted 模式 (測試前 Mock 創建)

```typescript
const mockInstallOrUpdateExtension: Mock = vi.hoisted(() => vi.fn());
const mockRequestConsent: Mock = vi.hoisted(() => vi.fn());

vi.mock('../../config/extension-manager.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import(...)>();
  return {
    ...actual,
    ExtensionManager: vi.fn().mockImplementation(() => ({
      installOrUpdateExtension: mockInstallOrUpdateExtension,
    })),
  };
});
```

### C. vi.spyOn 模式 (監視現有函數)

```typescript
let debugLogSpy: MockInstance;
let processSpy: MockInstance;

beforeEach(() => {
  debugLogSpy = vi.spyOn(debugLogger, 'log');
  processSpy = vi.spyOn(process, 'exit')
    .mockImplementation(() => undefined as never);
});

afterEach(() => {
  processSpy.mockRestore();
  debugLogSpy.mockRestore();
});
```

### D. HTTP Mocking (MSW)

```typescript
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

export const server = setupServer();

beforeAll(() => server.listen({}));

beforeEach(() => {
  server.resetHandlers(
    http.post(CLEARCUT_URL, () => HttpResponse.text())
  );
});

afterAll(() => server.close());
```

### E. 自訂 Matcher 模式

```typescript
function toHaveOnlyValidCharacters(this: Assertion, buffer: TextBuffer) {
  const invalidLines: Array<{ line: number; content: string }> = [];

  for (let i = 0; i < buffer.lines.length; i++) {
    if (invalidCharsRegex.test(buffer.lines[i])) {
      invalidLines.push({ line: i, content: buffer.lines[i] });
    }
  }

  return {
    pass: invalidLines.length === 0,
    message: () => `Expected valid characters but found invalid...`,
  };
}

expect.extend({ toHaveOnlyValidCharacters });

declare module 'vitest' {
  interface Assertion<T> {
    toHaveOnlyValidCharacters(): T;
  }
}
```

---

## 5. 整合 vs 單元測試分離

### A. 單元測試 (Packages)

**位置:** `packages/*/src/**/*.test.ts(x)`

**特徵:**
- 單一檔案聚焦一個模組/類別
- Mock 外部依賴 (fs, network, 其他服務)
- 使用 mocked 檔案系統隔離執行
- 快速執行 (< 30s 超時)

**範例結構:**
```typescript
describe('CommandService', () => {
  class MockCommandLoader implements ICommandLoader {
    loadCommands = vi.fn(async (): Promise<SlashCommand[]> => {...})
  }

  beforeEach(() => {
    vi.spyOn(debugLogger, 'debug').mockImplementation(() => {});
  })

  afterEach(() => {
    vi.restoreAllMocks();
  })

  it('should load commands from a single loader', async () => {
    const mockLoader = new MockCommandLoader([mockCommandA, mockCommandB]);
    const service = await CommandService.create([mockLoader], signal);

    expect(mockLoader.loadCommands).toHaveBeenCalledTimes(1);
    expect(service.getCommands()).toHaveLength(2);
  })
})
```

### B. 整合測試 (integration-tests/)

**位置:** `integration-tests/*.test.ts`

**特徵:**
- 透過 `TestRig` 完整 CLI 調用
- 真實檔案系統操作
- 模型回應 mocking (fake-responses.json goldens)
- 較長超時 (300s)
- 重試邏輯 (2 次重試)
- 完整工具執行流程

**範例:**
```typescript
describe('file-system', () => {
  let rig: TestRig;

  beforeEach(() => { rig = new TestRig(); })
  afterEach(async () => await rig.cleanup())

  it('should be able to read a file', async () => {
    await rig.setup('should be able to read a file', {
      settings: { tools: { core: ['read_file'] } },
    });
    rig.createFile('test.txt', 'hello world');

    const result = await rig.run({
      args: `read the file test.txt`,
    });

    const foundToolCall = await rig.waitForToolCall('read_file');
    expect(foundToolCall).toBeTruthy();
    validateModelOutput(result, 'hello world');
  })
})
```

### C. 測試分類摘要

| 方面 | 單元測試 | 混合整合 | 完整整合 |
|------|----------|---------|----------|
| 位置 | packages/*/src/**/*.test.ts | *.integration.test.ts | integration-tests/*.test.ts |
| 檔案系統 | Mocked (vi.mock('fs')) | 真實 (臨時目錄) | 真實 (TestRig.testDir) |
| 網路 | Mocked (MSW/vi.mock) | Mocked (MSW) | Mocked (fake-responses.json) |
| CLI 調用 | N/A | N/A | 完整透過 spawn/pty |
| 超時 | 30s | 30s | 300s |
| 重試 | 否 | 否 | 是 (2x) |
| 焦點 | 單一模組 | 配置/功能 | 端對端流程 |

---

## 6. 覆蓋率與 CI/CD 整合

### A. 覆蓋率配置

**Vitest 覆蓋率設置:**
```typescript
coverage: {
  enabled: true,
  provider: 'v8',
  reportsDirectory: './coverage',
  include: ['src/**/*'],
  reporter: [
    ['text', { file: 'full-text-summary.txt' }],
    'html',           // 可瀏覽的 HTML 報告
    'json',           // 機器可讀
    'lcov',           // 標準覆蓋率格式
    'cobertura',      // Jenkins/CI 格式
    ['json-summary', { outputFile: 'coverage-summary.json' }],
  ],
}
```

**覆蓋率輸出:**
- `coverage/` 每個套件的目錄
- `coverage-summary.json` - JSON 摘要
- `full-text-summary.txt` - CLI 友好文字
- `index.html` - 互動式 HTML 報告

### B. CI/CD 整合

**GitHub Actions 工作流程:** `/.github/workflows/ci.yml`

**測試 Jobs:**

1. **Lint Job (首先執行)**
   - ESLint, Prettier, yamllint, shellcheck, actionlint
   - 測試前的程式碼驗證

2. **Test Linux Job (矩陣)**
   ```yaml
   test_linux:
     runs-on: 'gemini-cli-ubuntu-16-core'
     strategy:
       matrix:
         node-version: ['20.x', '22.x', '24.x']
     steps:
       - Checkout
       - Set up Node.js ${{ matrix.node-version }}
       - Build project
       - Run tests: npm run test:ci
       - Upload junit test results
       - Upload coverage reports
   ```

3. **整合測試 (獨立 Job)**
   - 使用 `test:integration:sandbox:*` 變體
   - 使用 GEMINI_SANDBOX 環境變數執行
   - 支援 docker 和 podman 沙盒

### C. CI 環境變數

```bash
NO_COLOR=true                          # 一致的測試輸出
CI=true                                # test-helper 偵測
GEMINI_CLI_INTEGRATION_TEST=true       # globalSetup.ts 設置
GEMINI_FORCE_FILE_STORAGE=true         # CI 中無 keychain
GEMINI_SANDBOX={false|docker|podman}   # 沙盒模式
KEEP_OUTPUT={false|true}               # 保留測試產物
VERBOSE={false|true}                   # 額外日誌
```

**超時調整邏輯 (test-helper.ts Line 24-28):**
```typescript
function getDefaultTimeout() {
  if (env['CI']) return 60000;           // CI 中 1 分鐘
  if (env['GEMINI_SANDBOX']) return 30000; // 容器中 30s
  return 15000;                           // 本地 15s
}
```

---

## 7. 測試模式與最佳實踐

### A. 測試結構模式

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

  it('should do something specific', () => {
    // Arrange
    const input = 'test';

    // Act
    const result = functionUnderTest(input);

    // Assert
    expect(result).toBe('expected');
    expect(mockSpy).toHaveBeenCalledWith(input);
  });

  it.each([
    { input: 'a', expected: 'A' },
    { input: 'b', expected: 'B' },
  ])('should handle $input', ({ input, expected }) => {
    expect(functionUnderTest(input)).toBe(expected);
  });
});
```

### B. 整合測試模式

**模式 1: 檔案系統測試**
```typescript
it('should be able to read a file', async () => {
  await rig.setup('test name', {
    settings: { tools: { core: ['read_file'] } },
  });
  rig.createFile('test.txt', 'hello world');

  const result = await rig.run({
    args: `read the file test.txt`,
  });

  const foundToolCall = await rig.waitForToolCall('read_file');
  expect(foundToolCall).toBeTruthy();
});
```

**模式 2: 互動測試**
```typescript
it('should handle user input', async () => {
  const run = await rig.runInteractive();

  await run.expectText('Choose option:', 5000);
  await run.type('1');
  await run.expectText('Confirmed');

  const exitCode = await run.expectExit();
  expect(exitCode).toBe(0);
});
```

**模式 3: Hook 系統測試**
```typescript
it('should block tool execution with hook', async () => {
  await rig.setup('hook test', {
    settings: {
      tools: { enableHooks: true },
      hooks: {
        BeforeTool: [{
          matcher: 'write_file',
          hooks: [{
            type: 'command',
            command: 'node -e "console.log(...)"',
          }],
        }],
      },
    },
  });

  const result = await rig.run({ args: 'Create a file' });

  const toolLogs = rig.readToolLogs();
  const writeFiles = toolLogs.filter(t => t.toolRequest.name === 'write_file');
  expect(writeFiles).toHaveLength(0);
});
```

### C. 調試模式

```typescript
// 測試失敗時打印調試資訊
if (!foundToolCall || !result.includes('expected')) {
  printDebugInfo(rig, result, {
    'Found tool call': foundToolCall,
    'Contains text': result.includes('expected'),
  });
}

// 保留測試產物以供檢查
// KEEP_OUTPUT=true npm run test:e2e

// 使用詳細日誌執行
// VERBOSE=true npm run test:e2e

// 讀取原始工具日誌
const toolLogs = rig.readToolLogs();
console.log('All tools:', toolLogs.map(t => t.toolRequest.name));
```

---

## 8. 整合測試輔助框架

**檔案:** `/integration-tests/test-helper.ts` (1118 行)

### 核心類別

```typescript
export class InteractiveRun {
  ptyProcess: pty.IPty
  output: string

  async expectText(text: string, timeout?: number)
  async type(text: string)      // 逐字元輸入，等待回顯
  async sendText(text: string)  // 一次寫入整個字串
  async sendKeys(text: string)  // 逐字元無等待回顯
  async kill()
  expectExit(): Promise<number>
}

export class TestRig {
  testDir: string | null = null
  testName?: string

  // 設置方法:
  setup(testName: string, options?: {
    settings?: Record<string, unknown>
    fakeResponsesPath?: string
  })
  createFile(fileName: string, content: string)
  mkdir(dir: string)
  sync()  // 確保檔案系統同步

  // 執行方法:
  run(options?: {
    args?: string | string[]
    stdin?: string
    yolo?: boolean
  }): Promise<string>

  runCommand(args: string[], options?: { stdin?: string }): Promise<string>
  runInteractive(initialArgs?: string[]): Promise<InteractiveRun>

  // 結果檢查:
  readFile(fileName: string): string
  readToolLogs(): ToolLog[]
  getToolCallByName(toolName: string): ToolInvocation | undefined

  // 非同步等待:
  async waitForToolCall(toolName: string): Promise<ToolInvocation | undefined>
  async waitForAnyToolCall(toolNames: string[]): Promise<ToolInvocation | undefined>
  async waitForTelemetryEvent(eventName: string): Promise<boolean>

  // 清理:
  async cleanup()
}
```

---

## 摘要統計

- **測試檔案:** 541 個
- **單元測試套件:** 4 個 (cli, core, a2a-server, test-utils)
- **整合測試:** 1 個 (integration-tests/)
- **Vitest 配置:** 4 個 (不同超時: 30s, 300s)
- **Mock 工具:** 6 個專用工具
- **CI Node 版本:** 3 個 (20.x, 22.x, 24.x)
- **覆蓋率報告器:** 5 個 (text, html, json, lcov, cobertura)
- **整合測試沙盒模式:** 3 個 (none, docker, podman)
- **執行緒池:** 8-16 執行緒用於並行執行
- **整合測試重試:** 2 次自動重試
