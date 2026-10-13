# Playwright 自动化测试方案与底层架构全集深度剖析

## 1. 项目背景与系统架构全景

[Playwright](https://github.com/microsoft/playwright) 是微软开源的下一代跨浏览器端到端自动化测试平台。其架构彻底革新了传统 Selenium 的 WebDriver HTTP 轮询模式，采用全双工长连接直接接入 Chromium（CDP 协议）、Firefox（Juggler 协议）与 WebKit（Inspector 协议）的底层无头控制管道。

```
┌────────────────────────────────────────────────────────────────────────┐
│             Playwright Client (Node.js / Python / Java / .NET)         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ 结构化 JSON-RPC over stdio / WebSocket
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Playwright Driver Server (C++ / Node)               │
├───────────────────────────────┬────────────────────────────────────────┤
│ Browser Context Isolation     │ Network Route Interceptor              │
│ - 完全沙箱化的多 Context 隔离 │ - 全局双向拦截 HTTP/HTTPS 请求与响应   │
│ - Cookie/Storage/Cache 零污染 │ - 离线模式模拟、自定义网络瀑布流       │
├───────────────────────────────┼────────────────────────────────────────┤
│ Auto-Waiting & Locators Engine│ Artifacts Recorder Engine              │
│ - 可见性/可点击性/稳定度自旋  │ - Trace Viewer 毫秒级 DOM 快照录制     │
│ - 穿透 Shadow DOM 与 iFrame   │ - 屏幕录制视频与 Har 文件自动打包      │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Playwright 本身作为顶级自动化测试工具，其内部的自动化回归测试套件极为庞大且精密：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **CDP Protocol Wire Tests** | 底层 JSON-RPC 协议打包、帧拆解、跨进程心跳与重连 | Node.js 原生 Socket 测试 | 纯协议层数据包拦截与回放 |
| **Browser Context Isolation** | 多上下文并发运行、Cookie 跨 Context 隔离验证 | Chromium / Firefox / WebKit | 每次用例自动派发独立临时 Profile |
| **Auto-Waiting Locators** | 元素滚动吸附、遮挡穿透、动态禁用等待、Canvas 命中 | Playwright Test Runner | 构造包含动画延迟的极端 HTML 夹具 |
| **Network Interception Tests** | 请求 Abort、Mock 数据注入、Header 修改、离线模式 | HTTP/2 & HTTP/3 临时测试服务器 | 每套测试绑定动态未占用端口 |
| **Trace & Video Capture** | 录制 zip 打包、DOM 差分快照完整性、视频关键帧解码 | 真实系统文件读取与解压验证 | 临时沙箱文件目录，测试后物理删除 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：BrowserContext 沙箱隔离与 Cookie 存储零泄露测试

#### 3.1.1 业务功能与测试目标
Playwright 的核心优势是能在同一个受测浏览器实例中，以极快速度创建成百上千个互不影响的独立 `BrowserContext`。测试必须验证：
1. Context A 设置的 LocalStorage 与 Cookie 对 Context B 完全不可见。
2. Context 销毁时，所有关联的内存页与临时缓存被底层彻底清理。

#### 3.1.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('BrowserContext Sandboxing Suite', () => {
  test('strictly isolates cookies and localStorage between concurrent contexts', async ({ browser }) => {
    // 1. 创建第一个独立上下文并注入敏感 Session Cookie
    const contextA = await browser.newContext();
    const pageA = await contextA.newPage();
    await pageA.goto('https://example.com');
    await contextA.addCookies([
      { name: 'auth_token', value: 'secret_user_a', domain: 'example.com', path: '/' }
    ]);
    await pageA.evaluate(() => localStorage.setItem('user_pref', 'dark_mode'));

    // 2. 创建第二个独立上下文
    const contextB = await browser.newContext();
    const pageB = await contextB.newPage();
    await pageB.goto('https://example.com');

    // 3. 严格断言：Context B 完全无法读取 Context A 的任何持久化凭证
    const cookiesB = await contextB.cookies();
    expect(cookiesB.find(c => c.name === 'auth_token')).toBeUndefined();

    const localStorageB = await pageB.evaluate(() => localStorage.getItem('user_pref'));
    expect(localStorageB).toBeNull();

    // 4. 清理资源
    await contextA.close();
    await contextB.close();
  });
});
```

---

### 3.2 模块二：Auto-Waiting 定位器与穿透 Shadow DOM 测试

#### 3.2.1 业务功能与测试目标
现代前端框架大量使用 Web Components 和复杂的 CSS 动画。Playwright 的定位器具备自动等待（等待元素可点击、无动画、未被遮挡）与穿透 Shadow DOM 的能力。测试必须验证：
1. 深层嵌套 Shadow Root 内部的按钮能够直接穿透定位，无需手动展开。
2. 元素被透明遮罩覆盖时，点击操作能够自动重试等待遮罩消失，超时后抛出精准定位异常。

#### 3.2.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('Auto-Waiting and Shadow DOM Piercing Suite', () => {
  test('pierces open shadow roots and waits for overlay transition to finish', async ({ page }) => {
    // 注入一个带延迟消失遮罩与深层 Web Component 的测试 HTML
    await page.setContent(`
      <div id="host"></div>
      <div id="overlay" style="position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.5);"></div>
      <script>
        // 500ms 后自动淡出遮罩
        setTimeout(() => document.getElementById('overlay').remove(), 500);

        // 构造 Shadow DOM
        const host = document.getElementById('host');
        const root = host.attachShadow({ mode: 'open' });
        root.innerHTML = '<button id="inner-btn">Click Inside Shadow</button>';
        root.getElementById('inner-btn').onclick = () => { window.__clicked = true; };
      </script>
    `);

    // 1. 直接定位 Shadow DOM 内部元素并执行点击（自动等待遮罩消失）
    const innerBtn = page.locator('#host >> button#inner-btn');
    await innerBtn.click();

    // 2. 校验点击副作用真实生效
    const wasClicked = await page.evaluate(() => (window as any).__clicked);
    expect(wasClicked).toBe(true);
  });
});
```

---

### 3.3 模块三：全局网络路由拦截（Route Interceptor）与网络故障模拟测试

#### 3.3.1 业务功能与测试目标
测试必须验证 Playwright 拦截网络流量的能力：
1. 拦截目标 API 请求，修改响应 Header 并注入 Fake JSON。
2. 模拟网络异常中断（`route.abort('failed')`）以及模拟物理离线断网。

#### 3.3.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('Network Interception & Offline Simulation Suite', () => {
  test('intercepts API calls, injects fake response and tests offline fallback', async ({ page, context }) => {
    // 1. 配置路由拦截：篡改用户角色
    await page.route('**/api/user/profile', async (route) => {
      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({ name: 'Mocked Admin', role: 'SUPERUSER' }),
      });
    });

    await page.goto('https://example.com/profile');
    await expect(page.locator('[data-testid="user-role"]')).toHaveText('SUPERUSER');

    // 2. 模拟物理完全断网
    await context.setOffline(true);

    // 3. 校验网页离线横幅自动弹出
    await page.reload().catch(() => {}); // 捕获网络错误
    await expect(page.locator('text=You are currently offline')).toBeVisible({ timeout: 5000 });
  });
});
```

---

### 3.4 模块四：Trace Viewer 录制文件完整性与差分快照测试

#### 3.4.1 业务功能与测试目标
Playwright Trace 会在测试失败时打包生成 `.zip` 文件。测试必须保证：
1. 录制归档中包含每个动作的 Screencast 截图帧、Console 日志与 Network HAR。
2. 即使页面在崩溃边缘，Trace 文件也能被安全闭合，不产生损坏的不可读压缩包。

#### 3.4.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';
import fs from 'fs';
import path from 'path';

test.describe('Trace Artifact Recording Suite', () => {
  test('records full execution trace into readable zip archive', async ({ browser }) => {
    const context = await browser.newContext();
    await context.tracing.start({ screenshots: true, snapshots: true });

    const page = await context.newPage();
    await page.goto('https://example.com');
    await page.click('a');

    const tracePath = path.join(__dirname, 'test-trace.zip');
    await context.tracing.stop({ path: tracePath });

    // 物理验证：文件存在且大小 > 0
    expect(fs.existsSync(tracePath)).toBe(true);
    expect(fs.statSync(tracePath).size).toBeGreaterThan(1024);

    // 清理临时文件
    fs.unlinkSync(tracePath);
    await context.close();
  });
});
```

---

## 4. 测试工程化与基础设施

1. **多浏览器二进制矩阵安装管理**：
   * 在 CI 脚本中使用 `npx playwright install --with-deps` 安装 Chromium、Firefox 与 WebKit 的特定修正版本，杜绝系统底层共享库缺失。
2. **测试失败自动重放与 Trace 上传**：
   * 采用 `retries: 2` 机制，首次失败自动激活全量 Trace 录制，并将 `.zip` 归档上传至 GitHub Artifacts 以供追溯。

---

## 5. 对同类复杂系统的参考价值

* **深入协议层的自动化驱动模式**：淘汰传统的 HTTP 轮询模式，采用双向 WebSocket/JSON-RPC 直接接管内核，是实现毫秒级事件响应与零 Flaky 自动等待的必经之路。
