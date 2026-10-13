# Penpot 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Penpot](https://github.com/penpot/penpot) 是全球首个开源的企业级协同 UI/UX 设计与产品原型制作 Web 平台（Figma 的顶级开源替代品）。系统前端采用 ClojureScript + SVG/Canvas 混合渲染管线与 WebAssembly 数学几何加速库，后端采用 Clojure + PostgreSQL + Redis，基于全双工 WebSocket 构建毫秒级多用户协同设计引擎。

### 1.2 被测系统核心组成
1. **无限画布渲染与矢量引擎（Infinite Canvas & Vector Engine）**：处理数万个图形图层的视口缩放（Zoom）、平移（Pan）、布尔运算（Union, Intersect）与贝塞尔曲线变形。
2. **弹性自动布局引擎（Flex/Grid Auto-Layout Engine）**：在前端高频递归计算嵌套画框（Frames）的尺寸约束、填充对齐与间隙。
3. **多人协同与冲突解决（Multiplayer Collaboration Engine）**：基于操作转换（OT）或类似 CRDT 的模型，同步多光标轨迹、图层锁与几何属性合并。
4. **原型交互与变量动效（Prototyping & Interactions）**：画板之间连线转场、悬停触发动画与页面状态回放。

### 1.3 自动化测试的主要难点与挑战
* **画布交互脱离了标准 HTML 语义元素**：图形绘制在 SVG 视口或 Canvas 内，没有常规的 `<button>` 或 `<input>` 供测试定位，几乎全依赖屏幕绝对坐标与几何运算。
* **高频连续鼠标动作模拟精度要求极高**：绘制一条贝塞尔曲线需要连贯的 `mousedown -> mousemove -> mouseup` 复合动作，稍有抖动会导致控制点偏移。
* **多浏览器协同会话的并发时序同步**：自动化测试需要在一个测试用例内启动 2 个甚至 3 个并发浏览器上下文，模拟多人同时编辑同一画框。

---

## 2. 测试架构全景与分层策略

Penpot 团队建立了涵盖纯几何数学单元、协议同步与多实例画布 E2E 的完整测试网：

```
┌────────────────────────────────────────────────────────┐
│         Multi-Client Canvas Playwright E2E Suite       │  多人画布层：同时驱动两个 BrowserContext 协作画板
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Visual Regression & SVG Golden Snapshots       │  视觉回归层：比对复杂图形渲染出的 SVG 字节与图片像素
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Realtime Sync Protocol & Mutation Integration     │  协议协同层：验证图层变更操作（Mutations）广播与锁状态
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│        Clojure/Wasm Geometry & Matrix Unit Tests       │  几何计算层：纯算法验证二维变换矩阵、布尔运算与对齐
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：高精度画布鼠标拖拽绘制矩形与几何属性断言

#### 3.1.1 业务功能背景
用户选择矩形工具（按快捷键 `R`），在画布坐标 `(200, 200)` 按下鼠标左键，向右下拖动至 `(400, 350)` 并释放。系统必须在画布上创建对应的矩形图层，尺寸精确为 `宽 200px, 高 150px`，并在右侧属性面板显示正确的数值。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Playwright 画布坐标测试脚本）**：
```typescript
test('draw rectangle on canvas and verify computed geometric properties', async ({ page }) => {
  await page.goto('/#/workspace/file-123');
  const canvas = page.locator('#main-canvas');
  await expect(canvas).toBeVisible();

  // 1. 模拟按下快捷键 'R' 激活矩形工具
  await page.keyboard.press('r');

  // 2. 获取画布在当前视口的绝对偏移
  const canvasBox = await canvas.boundingBox();
  expect(canvasBox).not.toBeNull();

  const startX = canvasBox!.x + 200;
  const startY = canvasBox!.y + 200;
  const endX = canvasBox!.x + 400;
  const endY = canvasBox!.y + 350;

  // 3. 模拟连续鼠标拖拽绘制
  await page.mouse.move(startX, startY);
  await page.mouse.down();
  await page.mouse.move(endX, endY, { steps: 5 });
  await page.mouse.up();

  // 4. 断言底层数据结构：通过执行 evaluate 探测应用状态树（App-State）
  const createdShape = await page.evaluate(() => {
    // 获取当前选中的图层对象
    return window.penpotApp.getSelectedShapes()[0];
  });

  expect(createdShape.type).toBe('rect');
  expect(createdShape.width).toBe(200);
  expect(createdShape.height).toBe(150);

  // 5. 断言右侧检查器输入框数值联动
  await expect(page.locator('input[aria-label="Width"]')).toHaveValue('200');
  await expect(page.locator('input[aria-label="Height"]')).toHaveValue('150');
});
```

---

### 3.2 功能二：多人实时协同编辑与光标同步（Multiplayer Presence）

#### 3.2.1 业务功能背景
设计师 Alice 与设计师 Bob 同时在一个设计文件内作业。Alice 在画布上移动鼠标时，Bob 的屏幕上应实时出现标有“Alice”名字的彩色光标平滑移动；当 Alice 移动某个图形时，Bob 屏幕上的该图形必须同步位移。

#### 3.2.2 测试用例设计方案
* **测试用例实现（多 BrowserContext 并发协作测试）**：
```typescript
test('real-time presence and multi-user shape movement synchronization', async ({ browser }) => {
  // 1. 启动两套独立的上下文与页面
  const contextAlice = await browser.newContext();
  const contextBob = await browser.newContext();

  const pageAlice = await contextAlice.newPage();
  const pageBob = await contextBob.newPage();

  // 两人同时进入同一个设计画板
  await pageAlice.goto('/#/workspace/file-shared');
  await pageBob.goto('/#/workspace/file-shared');

  // 2. Alice 移动鼠标到画布特定坐标
  await pageAlice.mouse.move(500, 300);

  // 3. 在 Bob 的页面上断言看到 Alice 的协同光标标签
  const aliceCursorOnBobScreen = pageBob.locator('.remote-cursor:has-text("Alice")');
  await expect(aliceCursorOnBobScreen).toBeVisible();

  // 4. Alice 选中图层并向右平移 100 像素
  await pageAlice.click('.shape-layer[data-id="rect-1"]');
  await pageAlice.keyboard.press('ArrowRight'); // 移动一次

  // 5. 断言 Bob 屏幕上的该图层坐标实时发生改变，且不发生闪烁
  const rectOnBob = pageBob.locator('.shape-layer[data-id="rect-1"]');
  await expect(rectOnBob).toHaveAttribute('transform', /matrix\(.*100.*\)/);

  await contextAlice.close();
  await contextBob.close();
});
```

---

### 3.3 功能三：自动布局（Auto-Layout）嵌套尺寸动态反向传递

#### 3.3.1 业务功能背景
在一个设置为“垂直自动布局（Vertical Auto-Layout）”的外层画框内部，包含 3 个子元素。当用户将其中的文字元素由“短文本”改为“三行长文本”时，内层文字高度撑大，外层画框必须自动自适应延展，并将下方的兄弟元素自动下推。

#### 3.3.2 测试用例设计方案
* **设计策略**：
  * 修改内部子元素文本内容。
  * 触发自动布局求值事件。
  * 断言外层画框的 `height` 属性从原数值 `H` 自动膨胀为 `H + delta`，且紧邻的兄弟元素其 Y 轴坐标被精确下推 `delta` 像素。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **SVG 矢量快照比对（Golden Snapshots）**：
   * 将画布导出的 SVG 节点文本进行规范化（剔除自增随机 ID），与保存在代码仓库中的 `.svg` 金样比对，验证渐变色、阴影、描边滤镜未损坏。
2. **多进程 WebSocket 模拟服务器**：
   * 针对协同算法，除了浏览器 E2E 外，编写轻量级的模拟客户端群发突发变更，验证后端 Redis Pub/Sub 的吞吐与广播延迟。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **画布类系统测试必须打通“DOM 事件与内部状态树（App-State）”**：由于画布元素在 DOM 中极其扁平甚至是一块纯 Canvas，测试框架必须通过全局暴露受管的只读挂钩（如 `window.__TEST_HOOK__`），允许测试脚本直接查询核心几何图元的数据状态。
* **双上下文并发是验证“实时协同”的标准范式**：对于涉及在线文档、白板、协同表格的系统，不能只靠 Mock WebSocket，必须编写由单个测试驱动两个真实独立 BrowserContext 的并发脚本，检验端到端同步时效。
