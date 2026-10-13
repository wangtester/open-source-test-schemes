# Discourse 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Discourse](https://github.com/discourse/discourse) 是全球应用最广的现代化开源社区与协作平台（Stack Overflow 联合创始人 Jeff Atwood 发起）。系统后端基于 Ruby on Rails + PostgreSQL + Redis 构建，前端基于 Ember.js SPA，采用 MessageBus 实现全双工实时事件推送。

### 1.2 被测系统核心组成
1. **富文本编辑与即时排版（Composer & Markdown Engine）**：支持即时输入预览、文件拖拽异步上传、Onebox 链接卡片抓取与提及（@mention）自动建议。
2. **实时消息推送与长轮询（MessageBus & Live Updates）**：帖子点赞、在线打字状态提示（Typing indicator）、新回复毫秒级追加至客户端视图。
3. **内容审核、信誉系统与反作弊（Trust Levels & Akismet）**：根据用户在线时长、阅读量、点赞数自动化动态计算信任等级，并执行细粒度权限限制与自动化限流。
4. **后台高吞吐作业管道（Sidekiq & Distributed Locks）**：邮件批量派发、外部链接抓取缓存、定时统计汇总。

### 1.3 自动化测试的主要难点与挑战
* **高频实时通信的前端状态竞争**：多个客户端同时在同一主题下回复时，WebSocket/MessageBus 消息到达的乱序与 DOM 重绘同步。
* **极其复杂的多租户与权限矩阵（RBAC）**：分类读写权限、私密版块邀请、封禁状态与信任等级叠加后的权限越权风险。
* **富文本编辑器的光标与异步上传交互**：大文件切片上传、图片转码、Markdown 正则渲染与 XSS 过滤的综合测试。

---

## 2. 测试架构全景与分层策略

Discourse 构建了跨越十余年的经典大型 Web 系统测试防线：

```
┌────────────────────────────────────────────────────────┐
│           Playwright & QUnit Full-Stack E2E Suite      │  前端交互层：真实浏览器驱动富文本输入与实时推送
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         RSpec Request & Controller Integration Tests   │  接口集成层：全量路由、Rack 中间件与权限过滤器
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│          Model & Background Job Service Specs          │  数据处理层：ActiveRecord 事务、Sidekiq 队列与分布式锁
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Fabricators & Database Cleaner Harness         │  数据底座：标准化模型工厂（Fabrication）与数据库隔离
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：富文本 Composer 交互与文件拖拽异步上传

#### 3.1.1 业务功能背景
用户在发帖窗口输入内容，拖入一张图片后，前端应立即在光标位置插入 `![Uploading image.png...]()` 占位符，后台上传成功后自动无缝替换为真实 CDN URL 与图片尺寸哈希，不能破坏用户正在键入的文字。

#### 3.1.2 测试用例设计方案
* **测试用例实现（QUnit / Playwright 前端交互测试）**：
```javascript
test("drag and drop upload into composer updates markdown placeholder", async function (assert) {
  // 1. 打开 Composer 发帖编辑器
  await visit("/t/international-topic/1");
  await click("button.create-reply");
  assert.dom(".d-editor-input").isFocused();

  // 2. 模拟用户输入前置文字
  await fillIn(".d-editor-input", "Check this architecture diagram:\n");

  // 3. 构造伪造的图片 File 对象并触发 drop 事件
  const file = new File(["fake_image_bytes"], "diagram.png", { type: "image/png" });
  await triggerEvent(".d-editor-input", "drop", {
    dataTransfer: { files: [file] }
  });

  // 4. 断言中间状态：立刻呈现上传中占位符
  assert.dom(".d-editor-input").hasValue(/\[Uploading diagram\.png\.\.\.\]/);

  // 5. 等待后台上传 Mock 接口返回
  await settled();

  // 6. 断言终态：占位符被无缝替换为带宽高的真实 markdown 标签
  assert.dom(".d-editor-input").hasValue(/!\[diagram\|600x400\]\(upload:\/\/abc123xyz\.png\)/);
  // 预览区域同步完成 HTML 渲染
  assert.dom(".d-editor-preview img").exists();
});
```

---

### 3.2 功能二：MessageBus 实时推送与并发回复冲突检测

#### 3.2.1 业务功能背景
当管理员在后台修改某分类的访问权限时，已经打开该页面的普通用户前端应通过 MessageBus 实时接收到通道广播，页面自动淡出提示或重定向，防止用户继续在该分类下非法提交数据。

#### 3.2.2 测试用例设计方案
* **测试用例实现（RSpec 后端与长轮询集成测试）**：
```ruby
RSpec.describe "MessageBus Security and Channel Publishing", type: :request do
  fab!(:admin) { Fabricate(:admin) }
  fab!(:user) { Fabricate(:user) }
  fab!(:secure_category) { Fabricate(:category, read_restricted: true) }

  it "publishes secure events only to authorized client subscriber identities" do
    # 1. 模拟普通用户长轮询连接 MessageBus
    sign_in(user)
    post "/message-bus/#{user.id}/poll", params: { "/category/#{secure_category.id}" => 0 }
    
    # 2. 普通用户无权限订阅该安全分类通道，断言返回未授权状态
    expect(response.status).to eq(403)

    # 3. 管理员在后台操作触发消息发布
    sign_in(admin)
    messages = MessageBus.track_publish("/category/#{secure_category.id}") do
      secure_category.update!(name: "Top Secret Category")
    end

    # 4. 断言消息只写入了受保护通道，包含正确的序列号与数据有效载荷
    expect(messages.size).to eq(1)
    expect(messages.first.data["name"]).to eq("Top Secret Category")
  end
end
```

---

### 3.3 功能三：用户信任等级（Trust Level）自动晋升计算引擎

#### 3.3.1 业务功能背景
用户从“新手（TL0）”自动升级为“基本用户（TL1）”，必须严格满足：进入至少 5 个主题、阅读至少 30 个帖子、在线阅读时间累计超过 10 分钟。

#### 3.3.2 测试用例设计方案
* **设计策略**：通过模型夹具（Fabricator）精准构造累积时间戳与阅读记录，断言后台周期性任务 `PromotionService` 评估的幂等性与准确性。
* **测试用例代码**：
```ruby
RSpec.describe PromotionService do
  fab!(:user) { Fabricate(:user, trust_level: TrustLevel[0]) }

  it "promotes user to TL1 when all criteria are strictly satisfied" do
    # 模拟未达到阈值：仅阅读了 29 篇
    UserStat.where(user_id: user.id).update_all(
      topics_entered: 5,
      posts_read_count: 29,
      time_read: 601
    )
    PromotionService.new(user).review
    expect(user.reload.trust_level).to eq(TrustLevel[0])

    # 模拟补足第 30 篇阅读量
    UserStat.where(user_id: user.id).update_all(posts_read_count: 30)
    PromotionService.new(user).review

    # 断言成功晋升并记录系统操作审计日志
    expect(user.reload.trust_level).to eq(TrustLevel[1])
    expect(UserHistory.where(action: UserHistory.actions[:change_trust_level]).exists?).to be true
  end
end
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **多 Worker 并行数据库隔离（`parallel_tests`）**：
   * 后端通过 `parallel_split_test` 将几千个 RSpec 用例切分到 8 个进程，每个进程使用独立的 `discourse_test_1` 至 `discourse_test_8` 数据库，测试启动前由 template 数据库快速克隆。
2. **QUnit 前端资产预编译与无头 Chrome 驱动**：
   * 采用定制的 `rake qunit:test` 管道，自动编译 Ember Handlebars 模板，在虚拟无头 Chrome 中并行执行两千余个前端交互用例。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **将长轮询/WebSocket 测试抽象为消息追踪器（Track Publish）**：测试双向实时推送时，在后端包装消息捕获器，断言发布通道的权限过滤器比在前端盲目等待更具确定性。
* **富文本交互测试必须覆盖“占位符替换机制”**：对于图片、音视频等大文件异步上传，断言不仅要看最终 HTML，更要断言上传过程中的光标占位符能否稳定过渡，避免破坏输入状态。
