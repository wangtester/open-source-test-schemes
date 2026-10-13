# NextChat 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[NextChat](https://github.com/ChatGPTNextWeb/NextChat)（原 ChatGPT-Next-Web）是全球用户量最大的开源跨平台轻量级 AI 交互系统之一。项目在开发演进中重度结合 AI 辅助编程与自动化工程。系统采用 Next.js + React + Zustand 构建，支持 Web、PWA 以及基于 Tauri 的跨平台桌面端，其核心优势在于提供基于 WebDAV 与 UpStash 的去中心化、端到端加密状态同步。

### 1.2 被测系统核心组成
1. **多端数据同步引擎（Sync Engine）**：基于 WebDAV / UpStash 实现多设备间配置、面具（Masks）与会话历史的增量加密合并。
2. **三向合并与冲突解决算法（3-Way Merge & Conflict Resolver）**：解决两台设备在离线状态下各自产生新消息后的无锁智能合并。
3. **富文本与数学图表渲染管线**：支持 Markdown、KaTeX 数学公式、Mermaid 流程图、代码高亮即时渲染与防 XSS 注入。
4. **客户端本地缓存与导入导出**：本地 LocalStorage / IndexedDB 数据备份、格式迁移与自适应修剪。

### 1.3 自动化测试的主要难点与挑战
* **跨设备并发同步导致的数据覆盖丢失**：两端同时修改面具名称，如何保证同步过程不发生“后写覆盖先写（Lost Update）”。
* **动态 LaTeX / Mermaid 渲染报错导致页面白屏**：模型吐出残缺的公式或语法错误的流程图时，前端需优雅降级。
* **PWA 离线工作态与 ServiceWorker 缓存断言**：在网络断开时的离线访问稳定性。

---

## 2. 测试架构全景与分层策略

NextChat 建立了轻量极速的算法单测与网络仿真测试架构：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Mobile & Desktop E2E            │  跨端展示层：响应式视口、PWA 离线模式、导出功能测试
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Sync Engine & WebDAV Integration Tests            │  协议同步层：搭建虚拟 WebDAV 模拟器，验证加密上传拉取
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      3-Way Conflict Merge Algorithmic Tests            │  冲突合并层：纯数据结构断言多设备消息合并无损性
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Markdown & KaTeX Parser Unit Tests             │  渲染安全层：测试畸形输入、XSS 注入拦截与优雅降级
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：WebDAV 跨端三向合并（3-Way Merge）冲突解决断言

#### 3.1.1 业务功能背景
设备 A 在离线状态下创建了“会话 1”；设备 B 在离线状态下创建了“会话 2”。两台设备连网后先后触发 WebDAV 同步。合并算法必须将“会话 1”与“会话 2”全部保留并按时间戳重排，绝对不能发生设备 B 覆盖清空设备 A 数据的情况。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Vitest 纯算法单元测试）**：
```typescript
import { describe, it, expect } from 'vitest';
import { mergeSessions } from '@/app/utils/sync';

describe('Sync Engine: 3-Way Merge Reconciliation', () => {
  it('merges sessions from two independent devices without losing any conversations', () => {
    // 1. 基线会话列表（共同祖先）
    const base = [{ id: 'session-0', title: 'Old Topic', messages: [] }];

    // 2. 设备 A 新增 session-1
    const localA = [
      { id: 'session-0', title: 'Old Topic', messages: [] },
      { id: 'session-1', title: 'Topic from Device A', messages: [] }
    ];

    // 3. 设备 B 新增 session-2
    const remoteB = [
      { id: 'session-0', title: 'Old Topic', messages: [] },
      { id: 'session-2', title: 'Topic from Device B', messages: [] }
    ];

    // 4. 执行三向合并
    const merged = mergeSessions(localA, remoteB, base);

    // 5. 严格断言：合并结果必须包含 3 个会话，两台设备的新增均完整保留
    expect(merged).toHaveLength(3);
    const ids = merged.map(s => s.id);
    expect(ids).toContain('session-0');
    expect(ids).toContain('session-1');
    expect(ids).toContain('session-2');
  });
});
```

---

### 3.2 功能二：Mermaid / KaTeX 渲染异常优雅降级防白屏测试

#### 3.2.1 业务功能背景
模型如果生成了一段破损的 LaTeX 公式（如 `\frac{1}{` 缺失后半部分），前端渲染组件绝对不能抛出未捕获异常导致整屏崩溃，而必须优雅显示红色错误提示或直接回显原始文本。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```typescript
test('malformed LaTeX formula degrades gracefully without crashing UI', async ({ page }) => {
  await page.goto('/');
  // 注入包含语法损坏公式的 Mock 消息
  await injectMockMessage(page, 'Broken formula: $$\\frac{1}{$$ end.');

  // 1. 验证整个对话卡片仍然正常存在，无 React 崩溃 Error Boundary 弹窗
  await expect(page.locator('.chat-message')).toBeVisible();

  // 2. 断言渲染器退化显示文本或友好警示标签
  await expect(page.locator('.katex-error')).toBeVisible();
});
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **移动端与桌面端双视口 E2E 矩阵**：
   * 在 Playwright 中配置 `Pixel 5` 移动设备仿真与 1920x1080 桌面端，自动化断言侧边栏在手机端自动收起、滑动手势正常工作。
2. **轻量级内存 WebDAV 服务夹具**：
   * 在测试进程启动时快速挂载轻量 HTTP WebDAV 服务，验证端到端鉴权上传与拉取。

---

## 5. 对复杂系统的实战借鉴意义

* **客户端数据同步必须优先建立离线三向合并测试**：去中心化或多端同步系统，断网并发修改是家常便饭；必须把合并算法设计为纯函数并设计穷尽的笛卡尔积测试。
* **第三方渲染引擎必须有防御性测试套件**：引入富文本、Markdown、公式渲染库时，编写针对恶心格式的负向测试，是保证 Web 系统不闪退、不白屏的必备护栏。
