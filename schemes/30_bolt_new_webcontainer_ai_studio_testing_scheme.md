# Bolt.new 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[Bolt.new](https://github.com/stackblitz/bolt.new) 是基于浏览器的 AI 全栈全自动开发工作台。它革命性地结合了 LLM 自动化编程与 StackBlitz 独创的 **WebContainer**（浏览器沙箱微内核）技术，使得用户无需安装任何本地开发环境，即可在浏览器内部自举编译、安装 npm 依赖并启动 Node.js 服务。

```
┌────────────────────────────────────────────────────────────────────────┐
│               Bolt.new UI (Remix + Tailwind CSS + xterm.js)            │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ WebContainer API / ServiceWorker
┌───────────────────────────────────▼────────────────────────────────────┐
│                    WebContainer In-Browser Micro-OS                    │
├───────────────────────────────┬────────────────────────────────────────┤
│ Virtual File System (VFS)     │ Terminal & Process Runtime (xterm)     │
│ - 内存层叠文件系统 (Ext2-like) │ - 进程派生 (spawn) & exitCode 监听     │
│ - 监听文件变化 (watchers)     │ - 终端输入输出双工流 (stdin/stdout)    │
├───────────────────────────────┼────────────────────────────────────────┤
│ In-Memory Networking Stack    │ AI Code Diff / Action Stream Parser    │
│ - 虚拟端口监听 (port: 3000)   │ - 流式 XML / Markdown 动作块解析       │
│ - 内嵌浏览器 Iframe 热重载     │ - 自动转换为 VFS 写操作与执行指令      │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Bolt.new 采用以浏览器沙箱微内核为核心的分层自动化测试设计：

| 测试层级 | 被测目标与用例范围 | 技术选型与执行环境 | 隔离机制与关键策略 |
| :--- | :--- | :--- | :--- |
| **VFS Unit Tests** | 内存文件系统增删改查、层级目录创建、路径规范化 | Vitest (Node.js/JSDOM) | 纯内存虚拟文件映射 |
| **Stream Parser Tests** | AI 输出 XML 动作块（`<boltAction type="file">`）流式分片解析 | Vitest | 流式 chunk 切割注入（模拟慢速打字机） |
| **WebContainer Mock Tests** | 模拟容器内部 `npm install`、`vite dev` 进程输出流及退出码响应 | Vitest + Mock Process API | 避免真机下载几十兆 npm 包，秒级模拟完成 |
| **Terminal Integration** | xterm.js 键盘交互、ANSI 转义字符高亮渲染、进程中断（Ctrl+C） | Vitest + Testing Library | 模拟终端键盘事件捕获 |
| **Full Browser E2E** | 真实拉起 WebContainer 浏览器无头环境，执行从输入需求到 Iframe 出图全流程 | Playwright (Chromium/WebKit) | 隔离浏览器 Profile，配置 COOP/COEP 跨域隔离头 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：AI 动作流式解析器（Bolt Action Stream Parser）测试

#### 3.1.1 业务功能与测试目标
AI 在生成代码时，以流式传输 Markdown/XML 标记块（例如 `<boltArtifact id="project-01">` 与 `<boltAction type="file" filePath="src/App.tsx">`）。测试必须验证：
1. **分片切块容错**：即使 XML 标签被切断在两次网络数据包中（如 `<bolt` 和 `Action>`），解析器能自动缓存并正确识别完整边界。
2. **多文件并发提取**：连续生成多个文件动作时，解析器能顺序触发对应的写文件回调。

#### 3.1.2 测试用例实现
```typescript
import { describe, it, expect, vi } from 'vitest';
import { StreamingActionParser } from '~/lib/runtime/action-parser';

describe('Bolt Action Stream Parser Suite', () => {
  it('correctly parses fragmented XML action tags and dispatches file writes', async () => {
    const onFileAction = vi.fn();
    const onShellAction = vi.fn();

    const parser = new StreamingActionParser({
      onFileAction,
      onShellAction,
    });

    // 模拟 LLM 碎片化流式吐字（标签甚至被在属性中间割断）
    const chunks = [
      'I will now create the project.\n<boltAction type="',
      'file" filePath="src/index.ts">\nconsole.log("hello',
      ' world");\n</boltAction>\n',
      '<boltAction type="shell">npm install</bolt',
      'Action>',
    ];

    for (const chunk of chunks) {
      parser.feed(chunk);
    }
    parser.end();

    // 1. 断言文件动作被正确提取与装配
    expect(onFileAction).toHaveBeenCalledTimes(1);
    expect(onFileAction).toHaveBeenCalledWith({
      filePath: 'src/index.ts',
      content: 'console.log("hello world");\n',
    });

    // 2. 断言 Shell 动作被正确装配
    expect(onShellAction).toHaveBeenCalledTimes(1);
    expect(onShellAction).toHaveBeenCalledWith({
      command: 'npm install',
    });
  });
});
```

---

### 3.2 模块二：WebContainer 虚拟文件系统（VFS）内存操作与事件监听测试

#### 3.2.1 业务功能与测试目标
WebContainer 依赖虚拟文件系统管理整个项目源代码。测试必须覆盖：
1. 深度目录自动递归创建（`mkdir -p` 行为）。
2. 文件变更监听器（Watcher）在文件写入时及时发布准确的事件。
3. 试图非法写出工作区目录（`../../malicious.js`）的安全拦截。

#### 3.2.2 测试用例实现
```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { VirtualFileSystem } from '~/lib/runtime/vfs';

describe('Virtual File System (VFS) Suite', () => {
  let vfs: VirtualFileSystem;

  beforeEach(() => {
    vfs = new VirtualFileSystem();
  });

  it('creates parent directories recursively and emits change events', async () => {
    const watcherEvents: string[] = [];
    vfs.watch('/workspace', (event, path) => {
      watcherEvents.push(`${event}:${path}`);
    });

    // 1. 递归写入深层文件
    await vfs.writeFile('/workspace/src/components/Header.tsx', 'export const Header = () => null;');

    // 2. 读回校验内容
    const content = await vfs.readFile('/workspace/src/components/Header.tsx');
    expect(content).toBe('export const Header = () => null;');

    // 3. 验证目录节点是否真实存在
    expect(await vfs.exists('/workspace/src/components')).toBe(true);

    // 4. 监听器通知断言
    expect(watcherEvents).toContain('change:/workspace/src/components/Header.tsx');
  });

  it('rejects path traversal attempts outside workspace', async () => {
    await expect(async () => {
      await vfs.writeFile('/workspace/../../etc/passwd', 'root');
    }).rejects.toThrow(/Access denied|Path traversal detected/);
  });
});
```

---

### 3.3 模块三：终端模拟（xterm.js）与子进程命令执行控制测试

#### 3.3.1 业务功能与测试目标
当 WebContainer 运行 `npm run dev` 时，输出日志通过虚拟流注入到 xterm.js 中。测试必须保证：
1. ANSI 颜色字符与控制序列能被终端正确解析，避免输出乱码。
2. 进程派发（Spawn）正常捕获错误退出码（非零退出码）。

#### 3.3.2 测试用例实现
```typescript
import { describe, it, expect, vi } from 'vitest';
import { WebContainerProcessRunner } from '~/lib/runtime/process-runner';

describe('WebContainer Process Runner Suite', () => {
  it('captures process stdout, stderr and exitCode cleanly', async () => {
    const mockWebContainer = {
      spawn: vi.fn().mockResolvedValue({
        output: new ReadableStream({
          start(controller) {
            controller.enqueue(new TextEncoder().encode('\u001b[32mBuild complete!\u001b[0m\n'));
            controller.close();
          },
        }),
        exit: Promise.resolve(0),
      }),
    };

    const runner = new WebContainerProcessRunner(mockWebContainer as any);
    const logs: string[] = [];

    const exitCode = await runner.execute('npm', ['run', 'build'], (chunk) => {
      logs.push(chunk);
    });

    expect(exitCode).toBe(0);
    expect(logs.join('')).toContain('Build complete!');
  });
});
```

---

### 3.4 模块四：动态虚拟端口探测与内嵌 Iframe 联动测试

#### 3.4.1 业务功能与测试目标
在浏览器内部启动 Vite 后，Vite 会监听本地端口（如 5173）。WebContainer 发出 `server-ready` 事件，前端需要感知此事件并自动将右侧 Preview 视窗定向到内嵌虚拟 URL。

#### 3.4.2 测试用例实现
```typescript
import { describe, it, expect, vi } from 'vitest';
import { PreviewManager } from '~/lib/runtime/preview-manager';

describe('Preview Manager Port Detection Suite', () => {
  it('detects server-ready event and updates preview iframe target', () => {
    const onPreviewUrlChange = vi.fn();
    const manager = new PreviewManager({ onPreviewUrlChange });

    // 模拟 WebContainer 内部服务在 5173 端口就绪
    manager.handleServerReady(5173, 'https://localhost-5173.webcontainer.io');

    expect(onPreviewUrlChange).toHaveBeenCalledWith('https://localhost-5173.webcontainer.io');
    expect(manager.getActivePort()).toBe(5173);
  });
});
```

---

### 3.5 模块五：Playwright 端到端浏览器全栈沙箱（E2E）测试

#### 3.5.1 业务功能与测试目标
真实启动 Chromium 验证全流程：
1. 校验页面请求具有 `Cross-Origin-Embedder-Policy: require-corp` 与 `Cross-Origin-Opener-Policy: same-origin` 头部（WebContainer 运行必备先决条件）。
2. 输入提示词，验证 Monaco Editor 代码生成、Terminal 编译日志与 Preview 页面元素渲染三位一体。

#### 3.5.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('Bolt.new Full-Stack Browser E2E', () => {
  test('AI prompt generates code, boots Vite in WebContainer, and renders in iframe', async ({ page }) => {
    // 监听网络响应头是否满足 SharedArrayBuffer 安全限制
    page.on('response', (response) => {
      if (response.url().includes('localhost:5173')) {
        const headers = response.headers();
        expect(headers['cross-origin-embedder-policy']).toBe('require-corp');
        expect(headers['cross-origin-opener-policy']).toBe('same-origin');
      }
    });

    await page.goto('http://localhost:5173');

    // 1. 在主提示词框输入需求
    const promptInput = page.locator('textarea[placeholder*="How can Bolt help you today?"]');
    await promptInput.fill('Build a simple counter React application with increment button');
    await page.keyboard.press('Enter');

    // 2. 断言 Monaco 代码编辑器展开并写入 App.tsx
    const codeEditor = page.locator('.monaco-editor');
    await expect(codeEditor).toBeVisible({ timeout: 15000 });
    await expect(codeEditor).toContainText('useState(0)');

    // 3. 断言终端开始执行 vite
    const terminal = page.locator('.xterm-rows');
    await expect(terminal).toContainText('vite', { timeout: 20000 });

    // 4. 断言右侧预览视窗 (Iframe) 成功渲染 React Counter
    const previewIframe = page.frameLocator('iframe[title="preview"]');
    const counterButton = previewIframe.locator('button:has-text("Count: 0")');
    await expect(counterButton).toBeVisible({ timeout: 30000 });

    // 5. 在 Iframe 内发生实际交互，点击按钮验证前端状态响应
    await counterButton.click();
    await expect(previewIframe.locator('button:has-text("Count: 1")')).toBeVisible();
  });
});
```

---

## 4. 测试工程化与基础设施

1. **浏览器跨域隔离安全头配置**：
   * 在 Vite 测试服务器与 Playwright 测试配置中，必须硬性注入 `Cross-Origin-Embedder-Policy` 与 `Cross-Origin-Opener-Policy`，确保浏览器开启 `SharedArrayBuffer`，否则 WebContainer 无法派生 Web Worker 线程。
2. **纯内存 Mock 测试降本**：
   * E2E 测试因为需要下载 npm 依赖和编译消耗较多时间，因此 90% 的自动化用例集中在 Mock WebContainer Process 和内存 VFS，数秒即可跑完百项回归。

---

## 5. 对同类复杂系统的参考价值

* **WebContainer 类似架构的测试范式**：测试浏览器沙箱系统时，必须建立将“文件变更 -> 进程触发 -> 端口监听 -> 视窗渲染”四元组解耦断言的机制，既能单模块极速 Mock 验证，又能单链路真实闭环。
* **流式文本动作解构器测试**：大语言模型生成结果往往是不稳定和碎片化的，测试流式解析器时采用极端的分片切断用例（把标签字母切在两半）是杜绝线上白屏崩溃的关键。
