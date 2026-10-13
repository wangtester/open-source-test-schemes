# NodeBB 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[NodeBB](https://github.com/NodeBB/NodeBB) 是现代基于 Node.js 构建的高性能、实时全双工交互社区平台。系统后端采用 Node.js (Express + Socket.IO) 驱动，前端采用响应式模板与客户端动态水合技术，支持 Redis 或 MongoDB 作为底层持久化与 Pub/Sub 消息中枢。

### 1.2 被测系统核心组成
1. **全双工 WebSocket 通信总线（Socket.IO Event Hub）**：几乎所有交互（发帖、点赞、浏览量统计、在线状态）均通过 WebSocket 实时事件流在客户端与服务端之间双向流转。
2. **无限滚动动态数据流（Infinite Scrolling & DOM Virtualization）**：根据用户滚动坐标实时按需从后端分片拉取数据，并自动维护 URL 路径与历史堆栈。
3. **细粒度分类权限访问控制（Privileges & ACL Engine）**：基于角色（Admin, Moderator, Registered, Guest）与组别的多维度权限判定。
4. **可扩展 Hook 插件机制（Filter, Action, Static Hooks）**：允许第三方插件在请求生命周期中介入并修改数据结构。

### 1.3 自动化测试的主要难点与挑战
* **长连接全双工事件驱动的时序非确定性**：Socket 事件在网络抖动下的到达先后顺序难以预测，容易造成断言条件尚未就绪即判定失败。
* **双数据库引擎（Redis 与 MongoDB）的驱动适配**：底层数据结构在键值对存储与文档型存储上的表现差异。
* **高频滚动的滚动位置与动态加载断言**：用户快速滚动页面时，前端防抖与节流机制对自动化脚本的精准交互要求极高。

---

## 2. 测试架构全景与分层策略

NodeBB 演进出了以实时事件流和数据库适配器为核心的分层自动化测试体系：

```
┌────────────────────────────────────────────────────────┐
│             WebdriverIO / Playwright E2E Suite         │  前端交互层：真实页面模拟无限滚动、即时聊天与通知徽标
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Socket.IO Direct Event Integration Tests       │  双工协议层：利用 Socket.IO 客户端直接收发事件并断言
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Mocha & Chai REST / Controller Suite           │  业务集成层：覆盖用户认证、分类管理与权限 ACL 计算
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│       Database Abstraction Layer (Redis/MongoDB)       │  存储适配层：同一套数据层测试用例在不同数据库引擎上执行
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：基于 Socket.IO 客户端的即时通知与未读计数器测试

#### 3.1.1 业务功能背景
当用户 A 在帖子下 @ 用户 B 时，用户 B 的网页客户端应立即通过 WebSocket 收到 `event: 'notifications.get'` 消息，右上角铃铛图标的未读计数器从 0 变为 1，点击打开通知下拉菜单后，未读计数器自动清零并向服务端回传已读 ACK。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Mocha + Socket.IO Client 脚本）**：
```javascript
const io = require('socket.io-client');
const assert = require('assert');

describe('Realtime Notification Socket Dispatcher', () => {
  let socketUserB;

  before((done) => {
    // 1. 初始化用户 B 的长连接客户端
    socketUserB = io.connect('http://localhost:4567', {
      query: { token: 'AUTH_TOKEN_USER_B' },
      transports: ['websocket'],
    });
    socketUserB.on('connect', done);
  });

  after(() => {
    socketUserB.disconnect();
  });

  it('pushes notification in real-time when another user mentions User B', (done) => {
    // 2. 在客户端注册事件侦听器
    socketUserB.once('checkNotifications', (data) => {
      // 4. 断言通知有效载荷与计数
      try {
        assert.strictEqual(data.unreadCount, 1);
        assert.ok(data.latestNotification.bodyShort.includes('mentioned you'));
        done();
      } catch (err) {
        done(err);
      }
    });

    // 3. 模拟用户 A 调用发帖接口并进行提及
    apiUserA.post('/api/v3/topics', {
      title: 'Topic with Mention',
      content: 'Hey @userB please check this out!',
    });
  });
});
```

---

### 3.2 功能二：无限滚动（Infinite Scrolling）分页流与数据加载断言

#### 3.2.1 业务功能背景
当用户浏览一个包含 500 个帖子的长主题时，首屏只加载前 20 贴。当页面向下滚动超过当前视口 80% 时，前端自动静默请求接下来的 20 贴，并追加到 DOM 底部，同时浏览器地址栏无刷新变更 URL（如从 `/topic/1/1` 变更为 `/topic/1/25`）。

#### 3.2.2 测试用例设计方案
* **测试用例实现（Playwright E2E 交互测试）**：
```typescript
test('infinite scrolling loads next chunk and updates browser URL', async ({ page }) => {
  await page.goto('/topic/1/slug-long-topic');

  // 1. 断言首屏渲染帖子数量为 20
  await expect(page.locator('[component="post"]')).toHaveCount(20);

  // 2. 模拟向下滚动至页面底部（Scroll to Bottom）
  await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));

  // 3. 等待网络请求触发与新节点加载完成
  await page.waitForResponse(res => res.url().includes('/api/topic/pagination'));
  
  // 4. 断言新帖子被成功追加至页面（数量变为 40）
  await expect(page.locator('[component="post"]')).toHaveCount(40);

  // 5. 验证浏览器地址栏被 pushState 自动更新
  await expect(page).toHaveURL(/.*\/topic\/1\/slug-long-topic\/21.*/);
});
```

---

### 3.3 功能三：ACL 权限矩阵计算与缓存强失效（Cache Invalidation）

#### 3.3.1 业务功能背景
管理员在后台将“普通用户”在“内部讨论区”的发帖权限移除。此时由于高并发，权限通常被缓存于 Redis Hash 中。系统必须确保在权限修改的事务中，关联的权限缓存被严格原子化清空（Invalidate），防止出现“已撤销权限但由于缓存未过期仍可发帖”的安全漏洞。

#### 3.3.2 测试用例设计方案
* **测试编排**：
  1. 普通用户向目标版块发帖，返回 200，触发该用户的权限读缓存加载。
  2. 管理员调用 API 撤销其权限。
  3. 普通用户再次立即发起发帖请求。
  4. **硬性断言**：
     * 第二次请求立即返回 403 Forbidden。
     * 读取 Redis 中的 `privileges:cache:user:xxx`，断言缓存 Key 已被彻底驱逐或重新计算。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **双数据库矩阵并行测试（Dual-DB CI Matrix）**：
   * 所有后端 Mocha 测试集在 GitHub Actions 中分为两组：一组挂载 Redis 7 容器，另一组挂载 MongoDB 6 容器，两套流水线并行执行，确保数据库抽象层（Database Adapter）100% 兼容。
2. **连接孤儿回收与内存泄漏检测**：
   * 在 Mocha 的 `after` 钩子中，强制检查是否有悬挂的 Socket.IO 连接或未释放的定时器（利用 `wtfnode` 诊断工具打印挂起句柄）。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **将长连接协议测试与 UI 渲染测试解耦**：对于重度依赖 WebSocket/长轮询的系统，编写直接利用协议客户端（如 `socket.io-client`）收发数据的测试用例，比单纯在浏览器中等 UI 弹窗要稳定得多。
* **无限滚动交互必须验证“滚动条位置与历史 URL 同步”**：无限滚动不仅要测试“新数据能出来”，还要测试“浏览器前进后退、刷新时能否恢复到当前阅读进度的具体楼层”。
