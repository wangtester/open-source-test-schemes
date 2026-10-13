# AFFiNE 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[AFFiNE](https://github.com/toeverything/AFFiNE) 是全球领先的下一代开源“本地优先（Local-First）”多模态协同知识库与无界白板平台（Notion + Miro 的一体化开源替代品）。系统架构由高性能 **Rust 存储内核（OctoBase）**、TypeScript / React 前端、自研开源块编辑器引擎 **BlockSuite** 以及基于 **Yjs / CRDT** 的去中心化无冲突数据同步框架构成。

### 1.2 被测系统核心组成
1. **双模态无缝同构引擎（Page Mode & Edgeless Mode）**：同一份文档数据既可以作为结构化富文本“文档模式”展示，也可以一键切换为二维自由画布“无界白板模式”自由连线摆放。
2. **块级原子编辑器（BlockSuite & Block Tree）**：每个段落、代码块、数据库表格均为独立块节点，支持拖拽重排、嵌套缩进与快捷转换。
3. **本地优先与 CRDT 离线状态同步（Local-First & Yjs Sync）**：所有数据首先持久化在客户端 IndexedDB / SQLite，支持完全断网编辑；联网后与远端执行无锁 CRDT 状态向量合并（State Vector Merge）。
4. **Rust 原生持久化与协作后端（OctoBase）**：负责跨平台二进制数据编解码、多工作区同步与权限管理。

### 1.3 自动化测试的主要难点与挑战
* **双模态数据结构的转换保真度**：在文档模式中增加几行列表，切换到白板模式后，如何验证空间坐标自适应排版、连线与层级关系绝对不变形。
* **离线断网与网络重连冲突自愈测试**：如何在自动化脚本中逼真模拟网络断开、本地产生复杂变更、再次连网后双向 CRDT 合并的完整闭环。
* **富文本块级光标选择与拖拽事件的精确性**：多块框选、跨块复制粘贴、Tab/Shift+Tab 改变嵌套深度的键盘时序控制。

---

## 2. 测试架构全景与分层策略

AFFiNE 建立了自底向上的跨语言、多模态自动化测试矩阵：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Dual-Mode E2E Suite             │  前端交互层：驱动双模态切换、块级拖拽、离线断网编辑
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      BlockSuite Editor Component Tests (Playwright)    │  编辑器组件层：测试光标导航、Markdown 快捷输入、选区
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Yjs / CRDT Conflict Resolution Spec Tests         │  协同算法层：验证并发双向冲突分支合并后的收敛一致性
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Rust OctoBase Storage & Sync Engine            │  内核存储层：基于 cargo test 验证二进制快照与状态向量
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：文档模式与白板模式（Page <=> Edgeless）双模态切换保真度

#### 3.1.1 业务功能背景
用户在文档模式下输入三级标题与待办清单，点击顶部模式切换按钮切换至“无界白板模式”。系统必须自动将每个富文本段落映射为白板上的便签/画框节点，空间坐标合理排布；再切换回文档模式时，文字内容与层级结构 100% 还原。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Playwright E2E 交互测试脚本）**：
```typescript
test('dual-mode toggling preserves content structure and coordinates without data loss', async ({ page }) => {
  await page.goto('/workspace/all-pages');
  await page.click('button:has-text("New Page")');

  // 1. 在文档模式下录入带有层级的大纲内容
  const editor = page.locator('.affine-page-root');
  await editor.click();
  await page.keyboard.type('# Architecture Blueprint');
  await page.keyboard.press('Enter');
  await page.keyboard.type('- Core Engine');
  await page.keyboard.press('Enter');
  await page.keyboard.type('- Database Storage');

  // 2. 点击切换为白板模式（Edgeless Mode）
  await page.click('[data-testid="switch-edgeless-mode"]');
  await expect(page.locator('.affine-edgeless-root')).toBeVisible();

  // 3. 在白板模式中断言文字作为图形块节点（Surface Block）存在
  const canvasNode = page.locator('.edgeless-block-portal:has-text("Architecture Blueprint")');
  await expect(canvasNode).toBeVisible();

  // 4. 在白板上将该节点向右拖拽 200 像素
  const box = await canvasNode.boundingBox();
  await page.mouse.move(box!.x + 10, box!.y + 10);
  await page.mouse.down();
  await page.mouse.move(box!.x + 210, box!.y + 10, { steps: 5 });
  await page.mouse.up();

  // 5. 切换回文档模式，验证文字与层级完好无损
  await page.click('[data-testid="switch-page-mode"]');
  await expect(page.locator('.affine-page-root h1')).toHaveText('Architecture Blueprint');
  await expect(page.locator('.affine-list-block')).toHaveCount(2);
});
```

---

### 3.2 功能二：本地优先（Local-First）离线断网编辑与 CRDT 状态合并

#### 3.2.1 业务功能背景
用户在飞行模式（无网络）下打开 AFFiNE 网页，修改文档内容。系统利用本地 IndexedDB 存储变更。联网后，客户端自动将本地增量变更与云端发生的其他修改进行 Yjs 状态向量比对，实现无锁自动合并，绝不覆盖他人内容，也绝不丢弃本地输入。

#### 3.2.2 测试用例设计方案
* **测试用例实现（Playwright 网络离线仿真测试）**：
```typescript
test('offline local-first edits sync seamlessly upon network reconnection', async ({ page, context }) => {
  await page.goto('/workspace/doc-sync-test');
  await page.locator('.affine-paragraph').first().fill('Initial online text.');

  // 1. 模拟网络离线（Offline Simulation）
  await context.setOffline(true);

  // 2. 在离线状态下继续键入内容（依赖本地 IndexedDB 事务）
  await page.keyboard.press('Enter');
  await page.keyboard.type('Offline created paragraph.');

  // 3. 验证本地 DOM 立即响应渲染（Local-first 零时延原则）
  await expect(page.locator('.affine-paragraph:has-text("Offline created paragraph.")')).toBeVisible();

  // 4. 恢复网络连接（Online Reconnection）
  await context.setOffline(false);

  // 5. 拦截并断言 WebSocket 或同步接口发出的 CRDT Update 字节包
  const syncPacket = await page.waitForRequest(req => req.url().includes('/api/sync') || req.url().includes('/octobase'));
  expect(syncPacket.postDataBuffer().length).toBeGreaterThan(0);

  // 6. 刷新页面（重新从存储层拉取），验证两段文字均持久化成功
  await page.reload();
  await expect(page.locator('.affine-paragraph')).toHaveCount(2);
});
```

---

### 3.3 功能三：块级拖拽重排与树形层级调整（Drag & Drop Indent）

#### 3.3.1 业务功能背景
用户长按某个段落块左侧的“六点拖拽把手”，将其拖动至另一个父级块右下方，或者在块起始位置按下 `Tab` 键。系统应将其转换为子级节点，父子关系在 Yjs 文档树中动态重建。

#### 3.3.2 测试用例设计方案
* **设计策略**：
  * 通过键盘 `Tab` 改变缩进，断言块的 `parentFlavour` 和属性树中的 `index` 关系。
  * 模拟跨块剪切（`Cmd/Ctrl + X`）并粘贴（`Cmd/Ctrl + V`）至另一分支，断言底层 CRDT 树的无环性（Cycle-Free Tree Invariant）不受破坏。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Rust 与 WebAssembly 联合 CI 流水线**：
   * 后端通过 `cargo test --workspace` 对 OctoBase 执行单元测试，包括并发读写事务、WAL 日志截断与内存泄漏检查。
   * 前端自动化构建 WebAssembly 二进制胶水层，在无头 Chrome 中进行真机执行验证。
2. **多操作系统与 Electron 桌面端矩阵**：
   * 在 Web 端与桌面客户端（Electron）双通道上执行相同的测试用例集，检验跨平台本地 SQLite 数据库适配器的可靠性。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **本地优先（Local-First）系统必须配置“真实断网仿真测试”**：利用 Playwright 的 `context.setOffline(true)` 能够逼真验证客户端在断网、弱网、重连时的容灾表现，是检验离线缓存和状态机鲁棒性的利器。
* **双模态/多视图系统必须建立“同构不变性（Isomorphic Invariant）断言”**：当同一份底层数据存在两种完全不同的 UI 展现形态（如文档模式 vs 白板模式）时，在两种模式之间来回切换并校验数据树的不变性，是防止隐性数据腐化（Data Corruption）的黄金测试法则。
