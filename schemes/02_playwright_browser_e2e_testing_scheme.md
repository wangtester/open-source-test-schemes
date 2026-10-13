# Microsoft Playwright 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Microsoft Playwright](https://github.com/microsoft/playwright) 是现代端到端（E2E）Web 自动化测试的工业级标杆工具。它支持 Chromium、WebKit（Safari 内核）和 Firefox 三大主流浏览器引擎，覆盖 Node.js、Python、Java 和 .NET 多语言生态。

### 1.2 被测系统核心组成
1. **浏览器驱动与连接层（Browser Process & CDP/Bidi Driver）**：直接通过 CDP（Chrome DevTools Protocol）或自定义 Juggler 协议与浏览器底层二进制建立双工通道。
2. **页面抽象与 DOM 操作层（Page & Locator Engine）**：提供高级 DOM 选择器解析、动态元素智能重试定位、操作模拟（Click, Type, Drag）。
3. **上下文隔离层（BrowserContext）**：实现轻量级、毫秒级创建的多租户沙盒，完全隔离 Cookie、LocalStorage、Cache 与网络拦截规则。
4. **测试执行器与报告器（Playwright Test Runner）**：支持测试分片（Sharding）、并发 Worker 调度、自动化 Trace 录制、视频录制与 HTML 报告生成。

### 1.3 自动化测试的主要难点与挑战
* **UI 渲染与网络异步竞争（Flaky Tests）**：元素未渲染即点击、动画未结束即输入、Ajax 请求未返回即断言。
* **多浏览器引擎行为差异**：Chromium 与 WebKit 对 CSS/JS 标准的微小差异、剪贴板权限、弹窗策略不一致。
* **浏览器进程崩溃与资源死锁**：无头浏览器运行在高并发 CI 中极易出现 OOM、死锁或产生僵尸进程。

---

## 2. 测试架构全景与分层策略

Playwright 测试自身框架及其驱动能力时，采用了三层递进的测试架构体系：

```
┌────────────────────────────────────────────────────────┐
│             In-Browser Real Spec Matrix                │  真实浏览器矩阵：Chromium/WebKit/Firefox 多端执行
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│         BrowserContext & Protocol Mock Tests           │  协议通信层：CDP / WebSocket 帧抓包与契约断言
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│            Locator & Engine Unit Tests                 │  选择器引擎：纯逻辑选择器解析、AST 匹配与语法树验证
└────────────────────────────────────────────────────────┘
```

* **测试套件拆分**：
  * `page.spec.ts`：页面行为测试（导航、生命周期事件、弹窗监听）。
  * `locator.spec.ts`：定位器稳定性、严格模式、智能等待机制测试。
  * `network.spec.ts`：HTTP 拦截、请求修改、Mock 响应与 HAR 回放。
  * `trace.spec.ts`：执行追踪日志捕获与可观察性断言。

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：自动等待机制（Actionability Checks & Auto-waiting）

#### 3.1.1 业务功能背景
传统 Selenium 自动化测试中最臭名昭著的问题是 `ElementClickInterceptedException` 或 `StaleElementReferenceException`，通常需要大量 `sleep(3)`。Playwright 引入了可操作性自动等待（Attached, Visible, Stable, Receives Events, Enabled）。

#### 3.1.2 测试用例设计方案
Playwright 团队如何测试“等待机制自身是否正确”？
* **用例设计思路**：构建具有恶劣延迟条件的 HTML 页面夹具，断言测试框架能平滑等待，而在超时前绝不误报失败。
* **测试用例实现**：
```typescript
test('should wait for button to become enabled before click', async ({ page, server }) => {
  // 1. 搭建延迟变更的页面夹具：按钮初始为 disabled，1 秒后由 JS 动态启用
  await page.setContent(`
    <button id="target" disabled>Submit</button>
    <script>
      setTimeout(() => {
        document.getElementById('target').disabled = false;
      }, 1000);
    </script>
  `);

  // 2. 触发点击操作：测试框架应自动进入轮询等待
  const startTime = Date.now();
  await page.click('#target');
  const duration = Date.now() - startTime;

  // 3. 断言验证
  expect(duration).toBeGreaterThanOrEqual(950); // 证明没有立即报错退出
  expect(duration).toBeLessThan(3000);          // 证明状态就绪后立即触发，无额外滞后
});
```

---

### 3.2 功能二：BrowserContext 毫秒级环境隔离与状态泄漏防护

#### 3.2.1 业务功能背景
每个测试用例若都重新冷启动一次浏览器（Browser Launch），测试套件将极度缓慢（启动需数秒）。Playwright 采用单个浏览器实例下运行多个轻量级 `BrowserContext` 的方案。

#### 3.2.2 测试用例设计方案
* **验证目标**：确保两个并发或先后执行的 Context 之间，Cookie、IndexedDB 和 LocalStorage 完全隔离，杜绝跨用例状态污染。
* **测试用例实现**：
```typescript
test('contexts should have completely isolated storage states', async ({ browser }) => {
  // 创建两个独立的上下文
  const contextA = await browser.newContext();
  const contextB = await browser.newContext();

  const pageA = await contextA.newPage();
  const pageB = await contextB.newPage();

  // Context A 写入凭据
  await pageA.goto('http://localhost:3000');
  await pageA.evaluate(() => localStorage.setItem('token', 'AUTH_TOKEN_AAA'));
  await contextA.addCookies([{ name: 'session', value: '123', domain: 'localhost', path: '/' }]);

  // Context B 读取凭据
  await pageB.goto('http://localhost:3000');
  const tokenInB = await pageB.evaluate(() => localStorage.getItem('token'));
  const cookiesInB = await contextB.cookies();

  // 断言：Context B 完全无法读取 Context A 的任何存储
  expect(tokenInB).toBeNull();
  expect(cookiesInB).toHaveLength(0);

  // 清理
  await contextA.close();
  await contextB.close();
});
```

---

### 3.3 功能三：网络层透明拦截与 Mock（Network Routing）

#### 3.3.1 业务功能背景
E2E 测试中经常需要模拟后端接口 500、延迟 10 秒、或者直接注入假数据，验证前端界面的降级与容错。

#### 3.3.2 测试用例设计方案
* **测试用例实现**：
```typescript
test('should intercept API request and fulfill with mock data', async ({ page }) => {
  // 1. 设置网络拦截路由
  await page.route('**/api/v1/user-profile', async route => {
    // 检查请求参数
    expect(route.request().method()).toBe('GET');
    // 注入模拟响应
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ name: 'Mocked User', role: 'admin' }),
      headers: { 'X-Mock-Source': 'Playwright-Test' }
    });
  });

  // 2. 触发前端请求并断言 DOM 渲染
  await page.goto('http://localhost:3000/profile');
  await expect(page.locator('.username-badge')).toHaveText('Mocked User');
  await expect(page.locator('.role-tag')).toHaveText('admin');
});
```

---

### 3.4 功能四：Trace 录制与可观察性断言（Post-Mortem Debugging）

#### 3.4.1 业务功能背景
当 CI 中的测试用例在第 30 步偶然失败时，开发者如何复盘当时的真实屏幕、网络请求、DOM 快照与控制台日志？

#### 3.4.2 测试用例设计方案
* **用例设计机制**：在测试执行前后挂载 Tracing 钩子，不仅生成测试制品，还对生成的 `.zip` 追踪包内部结构进行解包验证。
* **断言设计**：
  * 解压 trace 包，验证包含 `trace.trace`（事件流）、`resources/`（网络截图与响应体）。
  * 断言追踪文件中记录了每一次操作（click, type）前后的 DOM 快照帧。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **分布式分片执行（Test Sharding）**：
   * 将庞大的用例集拆分至多台 CI 机器运行：`npx playwright test --shard=1/4`。
2. **Worker 异常崩溃隔离（Worker Isolation）**：
   * 每一个测试文件分配独立的 Node.js 子进程与 Browser 进程，若某个页面导致 Chromium 发生内核崩塌（Crash），仅该 Worker 重启，绝不串扰全局流水线。
3. **Flaky 自动重试与黄金视频归档**：
   * CI 开启 `retries: 2`；若重试通过，自动将初次失败的完整 Trace 包保留并标记为警告，通知开发人员排查。

---

## 5. 对复杂系统自动化测试设计的实战借鉴意义

* **将“操作前检查”内置于底层驱动**：设计业务层自动化框架时，不要让测试人员手动写 `waitForElementVisible`，应当在基础封装中统一集成“自动等待就绪”的拦截器。
* **利用沙箱上下文加速多租户测试**：对于 SaaS 或高隔离度系统，借鉴 `BrowserContext` 的隔离思想，在测试中快速创建租户沙盒，复用底层连接与单例，兼顾性能与纯净度。
* **网络拦截优于全链路后端打桩**：前端或端到端测试中，直接在传输层拦截接口，可以极低成本覆盖 401、403、502、网络超时等所有后端边缘场景。
