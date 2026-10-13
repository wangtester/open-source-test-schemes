# LobeChat 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[LobeChat](https://github.com/lobehub/lobe-chat) 是全球顶级的开源现代化 AI 个人工作台与插件交互平台。该项目在业界率先落地了 **“AI-First 智能体自举研发”** 模式，大量的 UI 组件、Zustand 状态流与单元测试均由 AI Agent 自动生成。系统采用 Next.js (App Router)、React 19、Zustand 响应式状态库，并支持基于 WebAssembly 的 **PGlite 浏览器内置关系型数据库**。

### 1.2 被测系统核心组成
1. **多模态与富交互会话树（Multimodal Chat & Branching Trees）**：支持文字、语音、视觉图片分析，会话支持在任意节点开辟多分支并行探索。
2. **插件生态与客户端沙箱执行（Plugin System & Function Calling）**：动态加载第三方 OpenAPI 插件，在浏览器安全沙箱内渲染可视化交互组件。
3. **Artifacts 独立交互代码沙箱**：类似 Claude Artifacts，实时在右侧抽屉渲染 HTML/SVG/React 独立运行态。
4. **PGlite 本地优先与云端多端同步**：通过 WebAssembly 在浏览器端运行原生 PostgreSQL，实现离线使用与云端同步。

### 1.3 自动化测试的主要难点与挑战
* **极其丰富的前端动效与响应式状态机**：Zustand store 包含几十个 slice，测试必须断言状态变更在无多余渲染的情况下正确触发。
* **Edge Runtime 流式协议响应**：Next.js Edge 路由流式传输在不同测试环境（Node vs Browser）下的 polyfill 差异。
* **插件动态 Schema 解析与组件挂载**：外部未知插件 Manifest 的容错校验与沙箱通信测试。

---

## 2. 测试架构全景与分层策略

LobeChat 建立了以 Vitest 为核心、覆盖组件与全链路 E2E 的自动化测试网：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Cross-Browser E2E Suite         │  端到端层：驱动会话输入、分支切换、插件市场交互与离线
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Zustand Store & State Slice Unit Tests (Vitest)   │  状态机层：纯内存断言多模态会话树的分支变动与撤销重做
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│       Edge Route & Streaming Protocol Handlers         │  接口协议层：测试 SSE 流式解析、Token 统计与错误降级
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│       PGlite Database Schema & Migration Tests         │  本地数据层：在内存中执行 SQL 脚本，验证表结构迁移
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：多分支会话状态树（Branching Conversation Tree）分支切换断言

#### 3.1.1 业务功能背景
用户对 Agent 的某一条回答点击“重新生成（Regenerate）”，系统生成新分支。用户可以在两条回答之间左右切换（`Branch 1/2`）。切换时，该回答之后的所有上下文关联必须自动切换至选定分支链条。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Vitest + Zustand Store 单元测试）**：
```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { useChatStore } from '@/store/chat';

describe('Chat Store Branching Mechanism', () => {
  beforeEach(() => {
    useChatStore.getState().clearAllSessions();
  });

  it('switches conversation branch and re-indexes downstream context', () => {
    const store = useChatStore.getState();
    const sessionId = store.createSession();

    // 1. 发送第一轮消息并生成两个不同分支回答
    store.sendMessage({ id: 'msg-1', content: 'Explain Quantum Physics' });
    store.receiveMessage({ id: 'ans-1-v1', parentId: 'msg-1', content: 'Answer Variant 1' });
    store.receiveMessage({ id: 'ans-1-v2', parentId: 'msg-1', content: 'Answer Variant 2' });

    // 2. 默认激活第一分支
    expect(useChatStore.getState().getActiveMessage('msg-1').content).toBe('Answer Variant 1');

    // 3. 触发分支切换动作
    store.switchMessageBranch({ messageId: 'msg-1', targetId: 'ans-1-v2' });

    // 4. 断言当前激活分支已被翻转
    expect(useChatStore.getState().getActiveMessage('msg-1').content).toBe('Answer Variant 2');
    // 断言导出的上下文扁平数组严格包含选中的分支内容
    const contextList = useChatStore.getState().getActiveContextList();
    expect(contextList.map(m => m.content)).toContain('Answer Variant 2');
    expect(contextList.map(m => m.content)).not.toContain('Answer Variant 1');
  });
});
```

---

### 3.2 功能二：插件 Manifest 动态校验与危险调用拦截

#### 3.2.1 业务功能背景
当 Agent 尝试调用一个插件时，若该插件声明的入参缺少必填属性，或插件返回了恶意的非标准 HTML，系统必须在客户端进行 Schema 拦截并友好提示。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
  * 构建一个参数违规的 Tool Call Payload。
  * 触发 `pluginRunner.execute(toolCall)`。
  * 断言：抛出受管的 `PluginValidationError`，不发起任何物理网络请求，且 UI 自动挂载“插件参数异常”告警卡片。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **i18n 自动化测试覆盖**：
   * 配备专用自动化用例扫描代码中的所有翻译 Key，断言在 20+ 种语言文件（zh-CN, en-US, ja-JP 等）中 100% 存在且无多余冗余键。
2. **多模型适配器回归矩阵**：
   * 使用 Fake LLM 模拟器对 OpenAI、Claude、Gemini、DeepSeek 等 15+ 种模型适配器进行同一套协议输入，断言返回标准化的统一流格式。

---

## 5. 对复杂系统的实战借鉴意义

* **复杂前端交互必须以“响应式 Store 单测”为防线重心**：在多分支、富客户端系统中，UI 渲染只是外壳，核心业务逻辑全在状态树中；用 Vitest 把 Store 测透，能解决 80% 以上的前端状态 Bug。
* **浏览器内嵌入式 SQL（PGlite）是离线测试的利器**：在测试本地优先系统时，利用 Wasm 版 Postgres 替代复杂的 Mock，可以让前端完全复用后端真实的 SQL 查询逻辑。
