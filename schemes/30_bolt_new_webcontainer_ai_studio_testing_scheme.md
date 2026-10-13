# Bolt.new 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Bolt.new](https://github.com/stackblitz/bolt.new) 是全球颠覆性的在浏览器端运行全自主 AI 全栈开发的 Web 应用（由 StackBlitz 团队开源）。其架构开创性地将 AI Agent 与浏览器端操作系统级沙箱 **WebContainer** 相结合。全栈采用 Remix / TypeScript 构建，在浏览器内直接运行完整的 Node.js、Vite 开发服务器、npm 包管理器与真实交互终端。

### 1.2 被测系统核心组成
1. **流式 Action 解析器（Streaming Action Parser & Artifact Dispatcher）**：实时解析大模型流式输出的自定义 XML 标签（`<boltArtifact>`, `<boltAction type="file">`, `<boltAction type="shell">`）。
2. **WebContainer 浏览器沙箱管理引擎**：在浏览器沙箱内启动 Node.js 进程、挂载虚拟文件系统并监听实时 Dev Server 端口。
3. **分栏工作区组件体系**：集成 Monaco 编辑器、xterm.js 虚拟终端以及 iframe 实时预览窗口。
4. **差量文件同步与热重载管道（HMR & Virtual File Sync）**：将生成的文件增量无缝推送到 WebContainer 内存文件系统中。

### 1.3 自动化测试的主要难点与挑战
* **边接收边解析的流式状态机容错性**：大模型流式返回时，标签经常被截断在中间（如 `<boltAction type="fil`），解析器必须平滑挂起并在下一块补齐时恢复。
* **WebContainer 对浏览器特权环境的严苛要求**：WebContainer 依赖 `Cross-Origin-Embedder-Policy: require-corp` 与 `Cross-Origin-Opener-Policy: same-origin`，自动化测试必须在定制 HTTP 头的无头浏览器中执行。
* **纯前端全栈的复杂进程治理**：在浏览器内部运行的 Vite 进程若挂死或端口冲突，测试需要感知并安全重启。

---

## 2. 测试架构全景与分层策略

Bolt.new 建立了“流式标记纯算法测试 + 真实浏览器 WebContainer 集成”的双层测试防线：

```
┌────────────────────────────────────────────────────────┐
│             Playwright WebContainer Full E2E           │  浏览器沙箱层：驱动启动真实 WebContainer、npm 与预览
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Streaming Action XML Parser Unit Tests         │  协议解析层：分块向状态机喂入残缺字符串，断言状态翻转
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│       Virtual File Tree & Diff Workbench Tests         │  文件系统层：验证虚拟内存文件读写与 Monaco 编辑器同步
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Terminal Output & Shell Command Codecs         │  终端通信层：测试 xterm.js ANSI 转义序列与退出码识别
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：流式 Action 标签切片容错解析器（Streaming Action Parser）测试

#### 3.1.1 业务功能背景
当模型返回内容时，解析器必须在收到 `<boltAction type="file" filePath="src/App.tsx">` 标签闭合时立刻在虚拟文件树中创建文件，并在后续流式内容涌入时不断向该文件追加内容，直到收到 `</boltAction>` 标记才闭合文件句柄。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Vitest 流式状态机测试）**：
```typescript
import { describe, it, expect } from 'vitest';
import { ActionParser } from '@/lib/runtime/action-parser';

describe('Streaming Action XML Parser', () => {
  it('correctly parses actions chunked across arbitrary byte boundaries', () => {
    const parser = new ActionParser();
    const actions: any[] = [];
    parser.on('action', a => actions.push(a));

    // 故意在任意位置切碎的输入流切片
    const chunks = [
      '<boltArtifact id="project-1" title="Demo">\n',
      '<boltAction type="fi',
      'le" filePath="src/index.ts">\n',
      'console.log("hello',
      ' world");\n',
      '</boltAction>\n',
      '<boltAction type="shell">\nnpm test\n</boltAction>\n',
      '</boltArtifact>'
    ];

    // 逐个分块灌入解析器
    for (const chunk of chunks) {
      parser.feed(chunk);
    }
    parser.end();

    // 断言解析出的 Action 集合
    expect(actions).toHaveLength(2);
    expect(actions[0].type).toBe('file');
    expect(actions[0].filePath).toBe('src/index.ts');
    expect(actions[0].content).toBe('console.log("hello world");\n');
    expect(actions[1].type).toBe('shell');
    expect(actions[1].content).toBe('npm test\n');
  });
});
```

---

### 3.2 功能二：Playwright 真实 WebContainer 浏览器端启动与端口监听断言

#### 3.2.1 业务功能背景
在真实浏览器中启动开发环境，Agent 生成并运行 `npm run dev`。WebContainer 内部的 Vite 服务器启动并在本地 `3000` 端口开启监听，外部 iframe 必须实时捕获该端口并在预览窗口渲染出真实的网页。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```typescript
test('boots WebContainer in Chromium, runs dev server, and loads iframe preview', async ({ page }) => {
  await page.goto('/workbench');

  // 1. 等待 WebContainer 在浏览器端初始化就绪
  await page.waitForSelector('.webcontainer-ready');

  // 2. 模拟派发执行 Vite 开发服务器
  await page.evaluate(async () => {
    await window.boltWorkbench.executeShell('npm run dev');
  });

  // 3. 监听 WebContainer 内部端口绑定事件
  await page.waitForSelector('iframe[data-preview-port="3000"]');

  // 4. 进入 iframe 内部断言页面渲染
  const previewFrame = page.frameLocator('iframe[data-preview-port="3000"]');
  await expect(previewFrame.locator('h1')).toHaveText('Vite + React');
});
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **COOP/COEP 安全响应头配置**：
   * 在运行 Playwright 测试的本地 Web 服务器中配置专用中间件，注入 `Cross-Origin-Opener-Policy` 头，使无头 Chromium 具备使用 `SharedArrayBuffer` 与 WebContainer 的权限。
2. **虚拟内存与浏览器崩溃监控**：
   * 在每个测试用例结束后，断言 WebContainer 的所有子进程已被杀死，避免残留孤儿 Wasm 进程占用浏览器渲染进程的堆内存。

---

## 5. 对复杂系统的实战借鉴意义

* **流式系统的测试核心在于“恶意分块灌入（Malicious Chunk Feeding）”**：任何边接收边处理的流式系统，单测中必须把数据在标签中间、引号中间、中文字符字节中间故意切断，验证状态机能否 100% 优雅过渡。
* **前沿 Web 技术必须坚持真实浏览器端集成（Real Browser E2E）**：对于涉及 WebAssembly、WebWorker、SharedArrayBuffer 的高阶前端系统，纯 JSDOM 无法模拟，必须通过 Playwright 驱动真实浏览器引擎进行全要素验证。
