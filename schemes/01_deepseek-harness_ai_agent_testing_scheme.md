# DeepSeek Harness 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
DeepSeek Harness (`dsh`) 是由 DeepSeek 开源的下一代自主 AI Agent 框架与运行宿主，采用 **“Everything is a Plugin”**（一切皆插件）的时空可组合架构（基于 Cordis 核心微内核）。系统由 TypeScript / Node.js 编写，支持 CLI、Web UI、Desktop 桌面客户端以及基于 ACP (Agent Client Protocol) 的跨进程通信。

### 1.2 被测系统核心组成
1. **AgentLoop（执行循环核）**：负责会话上下文加载、输入驱动（Inbox Claim）、Prompt 装配、LLM 轮次交互、流式分块处理与状态机转换。
2. **Tool Runtime & Sandbox（工具与沙箱系统）**：负责工具注册、参数校验、子进程隔离调用（如 Bash、文件读写、网络请求）以及安全边界控制。
3. **Session & Snapshot Engine（会话存储与快照层）**：负责会话事件序列化（JSONL 规范）、版本演进（v1/v2/v3 兼容迁移）、确定性回放投影。
4. **Transport & Gateway（网关与协议层）**：提供基于 HTTP/SSE、WebSocket、Stdio 的 ACP 协议跨进程与远程交互。
5. **Web / Desktop Client（用户交互端）**：基于现代前端与浏览器环境的高频交互界面。

### 1.3 自动化测试的主要难点与挑战
* **LLM 输出的随机性与高昂成本**：大模型调用存在非确定性输出、网络抖动、耗时久、Token 费用高的问题。
* **Agent 存在自我欺骗风险**：单纯断言 Agent 输出的日志或文本回复，可能出现 Agent 自称“执行成功”但实际系统文件未被写入的假阳性。
* **复杂异步子进程生命周期管理**：大量工具执行依赖后台子进程，容易造成进程残留、锁死、文件句柄泄露及 CI 资源耗尽。
* **跨平台文件系统与网络差异**：Windows 与 Linux 下的路径分隔符、换行符、并发文件锁与进程信号差异极大。

---

## 2. 测试架构全景与分层策略

DeepSeek Harness 建立了业界最严苛的自动化测试架构之一，其核心原则是：**“100% or it doesn't merge”（单文件 100% 分支与行覆盖门禁）**、**“Verify the world, not the self-report”（验证真实世界，而非自述日志）**、**“Prefer the real implementation over a mock”（仅在非确定性边界 Mock，内部管线全量真实装配）**。

```
                     ┌───────────────────────────────────┐
                     │   With-Key Real-API Smoke Test    │  高阶端到端：真实调用大模型，验证真实世界副作用
                     └─────────────────┬─────────────────┘
                                       │
                     ┌─────────────────┴─────────────────┐
                     │   Web Browser Snapshot (Playwright)│  前端 GUI 视觉与状态层：Chromium/WebKit 快照
                     └─────────────────┬─────────────────┘
                                       │
                     ┌─────────────────┴─────────────────┐
                     │   Session / ACP Snapshot Replay   │  回放层：消除时间戳后比对事件流序列
                     └─────────────────┬─────────────────┘
                                       │
                     ┌─────────────────┴─────────────────┐
                     │  Real-Composition Integration Test│  集成层：内置 Cordis 微内核真实挂载所有真实插件
                     └─────────────────┬─────────────────┘
                                       │
                     ┌─────────────────┴─────────────────┐
                     │  Unit & Property-Based Spec Tests │  单元层：快速执行、单文件 100% 覆盖率验证
                     └───────────────────────────────────┘
```

### 2.1 测试套件与运行配置分流
* **`vitest.config.ts`**：核心单元测试与集成测试，执行 1400+ 个 `.spec.ts` 文件，严格开启每文件 100% 行/分支/函数覆盖率门禁。
* **`vitest.snapshot.config.ts`**：快照对比套件，通过确定性引擎比对 JSONL、Markdown 输出。
* **`vitest.web.config.ts`**：基于真实 Chromium / WebKit 驱动的前端无头浏览器 E2E 测试。
* **`vitest.bench.config.ts` / `vitest.web-stress.config.ts`**：基准性能与前端高压测试。
* **`pytest.ini`**：跨语言 Python SDK 与运行时兼容性验证。

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：Agent 执行循环（AgentLoop）与状态机驱动

#### 3.1.1 业务功能背景
AgentLoop 是驱动智能体核心生命周期的引擎。负责轮询输入队列（Inbox）、装配 System Prompt、调用 LLM、调度 Tool 执行，并根据返回结果决定继续下一轮思考还是终止会话。

#### 3.1.2 测试用例设计方案
* **测试夹具设计（`packages/test-support/agent-loop-testkit`）**：
  不使用玩具 Mock，而是构建 `mountAgentLoopTestHarness`。真实挂载 `LlmRuntime`、`SessionStore`、`SessionProjectionRegistry`、`SystemPrompt`、`ToolRuntime`、`AgentRegistry` 等真实生产服务插件。
* **关键测试用例剖析**：
  1. **多轮次逐步推进用例（Step-by-Step Claim）**：
     * **场景**：用户在单轮或多轮会话中连续追加输入。
     * **输入**：向 `DriverInbox` 注入多个 `UserMessage`，调用 `harness.claim(agent, target, turn)`。
     * **断言**：断言返回的消息顺序严格满足“下一步指令在前，换轮指令在后”的拓扑顺序；状态机从 `IDLE` -> `RUNNING` -> `WAITING_FOR_TOOL` -> `IDLE` 正常翻转。
  2. **会话中断与取消（Cancellation Mid-Stream）**：
     * **场景**：LLM 正在流式生成或工具正在执行时，客户端下发 Cancel 信号。
     * **断言**：未完成的 Tool 调用不应触发回调，未完成的 LLM 流切片不会落盘到 session JSONL 中，会话安全回退到上一个确定的检查点。

---

### 3.2 功能二：网络与 LLM 故障注入（Scriptable LLM Wire Fault Server）

#### 3.2.1 业务功能背景
生产环境中，大模型 API 经常出现 HTTP 429（限流）、500/503 服务端异常、SSE 流中途静默断开（EOF）、超长 Token 截断、JSON 格式损坏等极端情况。如果未针对异常设计恢复策略，Agent 会卡死或产生脏状态。

#### 3.2.2 测试用例设计方案
系统自研了专用的 Mock HTTP/SSE 服务：`@deepseek-ai/dsh-llm-mock-server`，支持脚本化预置 24 种行为切片（`MOCK_LLM_BEHAVIORS`）。

#### 3.2.3 关键用例与注入设计：
```typescript
// 典型用例配置：通过脚本编排一次具体的异常恢复测试
const mockBehaviors: MockLlmBehavior[] = [
  'connection_reset',      // 第1次请求：TCP连接直接被重置
  'rate_limit',            // 第2次请求：模拟 HTTP 429 速率限制
  'partial_disconnect',    // 第3次请求：流式吐出一半文本后网络异常断开
  'success',               // 第4次请求：正常响应并完成回复
];
```

* **用例覆盖场景**：
  1. **TCP Connection Reset（连接重置测试）**：
     * **测试操作**：Server 在接收到请求头后立即调用 `socket.destroy()`。
     * **测试断言**：断言 Client 自动触发指数退避重试（Exponential Backoff），请求计数器自增，会话不抛未捕获异常。
  2. **Partial Stream Disconnect（流切片中途断开测试）**：
     * **测试操作**：Server 正常发送 `event: message` 的前 2 个 chunk，随后主动切断 SSE 管道且不发送 `[DONE]` 标记。
     * **测试断言**：断言 Agent 能够识别出不完整的数据帧，丢弃残缺的分块，重新发起带已有上下文的续传或整轮重试，不会在持久化日志中留下破损的 AST。
  3. **Malformed JSON / Event（协议解析容错测试）**：
     * **测试操作**：Server 返回非合法的 SSE 格式文本或残缺的 JSON 字符串。
     * **测试断言**：底层反序列化捕获语法错误并包装为受管错误类型，触发重试或友好降级，防止进程崩溃。

---

### 3.3 功能三：工具沙箱隔离与子进程生命周期（Sandbox & Process Plumbing）

#### 3.3.1 业务功能背景
Agent 需要在本地执行 Shell 命令、文件修改。如果测试中对临时文件和子进程管理不当，不仅会产生 CI 假红（Flaky），还会导致磁盘空间被占满。

#### 3.3.2 测试用例设计方案
* **设计原则**：`Verify the world, not the self-report`（验证真实世界，而非自述日志）。
* **关键测试用例剖析**：
  1. **真实文件写入与字节对齐断言**：
     * **设计模式**：测试用例向 Agent 发送创建文件的 Prompt，Agent 内部调用 Tool 写入文件。
     * **断言方式**：**严禁仅检查 Agent 返回的“已创建文件成功”文本**。测试用例直接使用 `fs.readFileSync` 读取磁盘目标路径，比对文件权限、文件内容字节哈希；同时验证工作区外的受保护文件字节数严格不变（未受污染）。
  2. **孤儿进程清理与超时预算（Subprocess Teardown & Lifetime）**：
     * **设计模式**：采用 `execa` 封装子进程，测试启动长轮询或僵死命令（如 `sleep 1000`）。
     * **断言方式**：在 `afterEach` 生命周期钩子中强制检查进程树 PID，验证当上下文 dispose 时，进程信号能够正确下发到整个进程组（Process Group Kill），无孤儿进程残留。

---

### 3.4 功能四：确定性会话快照与回放（Deterministic Session Snapshot）

#### 3.4.1 业务功能背景
当重构 Agent 逻辑或升级 Prompt 模板时，如何确保历史数百种交互场景没有产生破坏性回退（Regression）？

#### 3.4.2 测试用例设计方案
采用 **Record & Replay + Envelope Projection（时空包络投影）** 机制。
* **消除时间与随机数差异**：
  会话 JSONL 文件中记录了所有 Event。但在快照比对时，原始的时间戳（Timestamp）、随机生成的 UUID、请求耗时每次都不同。
* **用例设计实现**：
  1. **Record 模式**：本地跑一次生成金样（Golden Fixture），包含 `input.json`、`session.jsonl`、`stdout.expected.jsonl`。
  2. **Replay 模式**：测试运行器通过虚拟时钟（Deterministic Clock）介入，将生成的时空包络（Sequence/Time Envelope）进行规范化擦除（Sanitize/Projection），只比对因果依赖图和语义字段。
  3. **向下版本兼容断言**：保留历史 `session.v1.jsonl`、`session.v2.jsonl`，运行当前最新版本的反序列化引擎，断言能够 100% 正确升级为 `session.v3` 内存模型。

---

### 3.5 功能五：真实发布制品入口验证（Real Entry Path & Built-Bin Smokes）

#### 3.5.1 业务功能背景
很多项目在开发时使用 `tsx` 或 TypeScript 实时加载，单元测试全部通过，但打包编译成 `dist/lib` 后由于打包配置、缺失依赖或模块规范冲突（ESM 与 CJS 混用）导致线上运行崩溃。

#### 3.5.2 测试用例设计方案
* **设计模式**：配置专门的 `built-bin.e2e.ts` 测试。
* **执行方式**：
  * 先运行真实构建编译出 `lib/bin.js`。
  * 测试用例通过原生 `node ./lib/bin.js` 启动真实子进程（脱离任何开发态 tsx 包装）。
  * 传入 `--help`、`--version` 以及非法参数。
  * 外部检查退出码 `exitCode === 0` 或指定非零退出码，检查 stderr 错误流中的格式化报错。

---

## 4. 自动化测试基础设施与 CI 质量门禁体系

### 4.1 100% 覆盖率门禁（Per-File 100% Coverage Gate）
* **规则**：`statements: 100, branches: 100, functions: 100, lines: 100`。
* **防作弊机制**：必须按文件独立计算（`perFile: true`）。不允许一个 10,000 行高覆盖的核心文件去“摊薄”一个仅有 50% 覆盖的边缘文件。
* **例外治理**：如果某行代码确实无法在纯内存中覆盖（如需要真正的浏览器 GPU 硬件），必须写明带有明确 Issue 跟踪的注释，并在配置中精确排除该特定行。

### 4.2 Flaky Test 预算与并行隔离控制
* **资源归属原则**：Vitest 采用多 Worker 并行派发。所有用例必须分配独立随机端口、独立临时目录（`tmpdir/dsh-test-${uuid}`）。
* **无状态清理**：`afterEach` 与 `afterAll` 必须保证在超时或断言失败时依然能够执行清理，防止污染同 runner 的下一个用例。

---

## 5. 对复杂系统自动化测试设计的实战借鉴意义

1. **针对 AI / LLM 系统的测试架构**：
   * 采用“三层漏斗”：轻量单元测试（Mock边界） -> 故障注入测试（编排Mock Server模拟异常网络） -> 关键路径真实 API 烟雾测试（With-Key 探针）。
2. **拒绝“自述断言”**：
   * 任何涉及 IO、持久化、进程调用的功能，一律断言物理世界（磁盘文件内容、网络请求抓包、操作系统进程状态），而非系统打印的 Success 日志。
3. **消除时间与环境非确定性**：
   * 在快照与回放测试中引入虚拟时钟与包络投影，使得分布式/事件溯源系统的快照比对具备 100% 确定性。
4. **编译制品烟雾回归（Built-Artifact Smoke）**：
   * 将构建后产物的原生入口测试纳入常规流水线，杜绝“代码测试全绿，打包发布即崩”的发布陷阱。
