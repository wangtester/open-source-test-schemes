# Appwrite 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Appwrite](https://github.com/appwrite/appwrite) 是全球知名的开源后端即服务（BaaS）平台（Firebase 的主流开源替代品）。系统后端采用高并发常驻内存的 PHP Swoole 协程框架驱动，前端提供基于 Svelte / React 构建的现代化开发者管理控制台，数据底座依托 MariaDB、Redis 以及 Docker 容器化编排。

### 1.2 被测系统核心组成
1. **多租户控制台（Developer Console）**：可视化项目管理、API Key 发行、Webhooks 配置与团队权限分配。
2. **文档数据库引擎（Database & Collections）**：支持强类型 Schema 校验、行级/文档级权限（Document Security）、索引与全文检索。
3. **云函数沙箱（Serverless Functions Runtime）**：利用独立 Docker 容器沙箱安全执行用户编写的 Node.js/Python/Go 代码。
4. **实时事件订阅通道（Realtime WebSocket Engine）**：基于 Swoole WebSocket 长连接，实现数据库增删改实时消息推送。

### 1.3 自动化测试的主要难点与挑战
* **微服务拓扑高度复杂**：Appwrite 包含 Core API、Realtime、Usage Tracker、Task Workers 等十几个微服务容器，测试必须在全拓扑协同下运行。
* **常驻内存与协程泄漏风险**：Swoole 进程不重启，测试用例若残留全局变量或未关闭的协程，会导致后续用例随机失败。
* **云函数沙箱容器的冷启动与销毁**：自动化测试需要验证容器启动超时、代码挂死时的自愈能力。

---

## 2. 测试架构全景与分层策略

Appwrite 构建了基于 Docker Compose 容器编排的全链路集成测试体系：

```
┌────────────────────────────────────────────────────────┐
│             Console Web UI Playwright Suite            │  控制台交互层：验证项目创建、Schema 拖拽配置、成员邀请
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│         Realtime WebSocket Protocol Tests              │  双工通信层：验证数据库变动事件向长连接客户端广播
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Swoole Async REST API Integration Suite        │  接口业务层：覆盖 Auth、Database、Storage、Functions
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│     Docker Compose Complete Topology Testbed           │  环境底座：启动包含 Redis/MariaDB/Appwrite 真实微服务集群
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：文档级细粒度权限（Document Security & Permissions）

#### 3.1.1 业务功能背景
集合开启了 `Document Security`。用户 A 创建了一个私密文档，设置权限仅允许 `read("user:userA")`。用户 B 登录系统发起检索，哪怕传入该文档的真实 ID，系统也必须返回 404（对未授权实体隐匿存在性）而不是 200。

#### 3.1.2 测试用例设计方案
* **测试用例实现（PHP / Swoole 测试客户端）**：
```php
public function testDocumentLevelSecurity()
{
    // 1. 用户 A 登录并创建包含专属权限的文档
    $clientA = $this->createAuthenticatedClient('userA@example.com');
    $database = new Database($clientA);
    $doc = $database->createDocument('my_database', 'secret_notes', 'ID_NOTE_1', [
        'content' => 'Top secret content for User A only'
    ], [
        Permission::read(Role::user('userA')),
        Permission::write(Role::user('userA'))
    ]);
    $this->assertEquals(201, $doc['$statusCode']);

    // 2. 用户 B 登录发起获取该文档的请求
    $clientB = $this->createAuthenticatedClient('userB@example.com');
    $databaseB = new Database($clientB);
    
    // 3. 断言返回 404 Not Found（严格安全规范：不返回 403 以防泄露文档存在性）
    try {
        $databaseB->getDocument('my_database', 'secret_notes', 'ID_NOTE_1');
        $this->fail('User B should not be able to read User A document');
    } catch (AppwriteException $e) {
        $this->assertEquals(404, $e->getCode());
        $this->assertStringContainsString('document_not_found', $e->getType());
    }
}
```

---

### 3.2 功能二：云函数容器沙箱执行与超时熔断

#### 3.2.1 业务功能背景
用户上传了一段恶意的无限循环死循环代码（如 `while(true) {}`），设置云函数超时时间为 3 秒。系统必须在第 3 秒准时向执行容器下发 `SIGKILL`，返回状态 `failed` 并正确收集报错日志，不能导致宿主机 CPU 耗尽。

#### 3.2.2 测试用例设计方案
* **测试用例编排**：
  1. 上传测试代码包（包含死循环的 `index.js`），设置 `timeout = 3`。
  2. 触发一次同步执行 `functions.createExecution(funcId)`。
  3. 记录请求总耗时。
  4. **断言核心指标**：
     * 请求总耗时严格控制在 3000ms - 3500ms 之间。
     * 执行记录中 `status` 为 `failed`，`errors` 字段包含 `Execution timed out`。
     * 宿主机检查后台 Docker 容器，断言执行沙箱容器已被立即销毁并回收。

---

### 3.3 功能三：实时 Realtime WebSocket 双工广播断言

#### 3.3.1 业务功能背景
前端客户端通过 WebSocket 订阅 `channels: ["databases.main.collections.chat.documents"]`。当另一个客户端通过标准 RESTful 接口创建了一条新消息时，长连接必须在毫秒级内收到包含新文档全量字段的 Push 广播。

#### 3.3.2 测试用例设计方案
* **测试实现步骤**：
```typescript
test('realtime client receives event when REST API creates document', async () => {
  // 1. 建立真实的 WebSocket 客户端连接并订阅频道
  const ws = new WebSocket('ws://localhost/v1/realtime?project=test_project&channels[]=databases.main.collections.chat.documents');
  const messagePromise = new Promise(resolve => {
    ws.on('message', data => resolve(JSON.parse(data.toString())));
  });

  // 2. 模拟第三方客户端通过 HTTP REST 接口创建文档
  await api.post('/v1/databases/main/collections/chat/documents', {
    documentId: 'msg_001',
    data: { text: 'Hello Realtime' }
  });

  // 3. 拦截 WebSocket 广播包并断言
  const event = await messagePromise;
  expect(event.events).toContain('databases.main.collections.chat.documents.*.create');
  expect(event.payload.$id).toBe('msg_001');
  expect(event.payload.text).toBe('Hello Realtime');
  
  ws.close();
});
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **容器化健康检查门禁（Docker Healthchecks）**：
   * 在 CI 运行测试前，测试启动脚本持续轮询所有 15 个容器的 `/v1/health` 探针，直至所有容器均为 `healthy` 才开始执行用例，杜绝冷启动未就绪导致的 Flaky。
2. **多语言 SDK 自动化代码生成与回归测试**：
   * 每次主干变更，自动生成 Web、Flutter、iOS、Android、Node 等 10 种 SDK，并由各自的自动化测试用例连接集成测试集群完成全量回归。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **细粒度权限必须测试“信息隐匿性（Information Hiding）”**：在测试高安全要求系统时，无权限访问的数据不应返回 403（让攻击者知道资源存在），而应断言返回 404，在安全边界上做极致测试。
* **对于沙箱/外部插件执行必须具备“超时熔断硬断言”**：不可信代码的执行测试，重点是验证在死循环、大内存占用、网络受限等极端场景下系统能否安全兜底并回收物理资源。
