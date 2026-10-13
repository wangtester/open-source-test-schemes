# Temporal 自动化测试方案与分布式工作流用例全集深度剖析

## 1. 项目背景与系统架构全景

[Temporal](https://github.com/temporalio/temporal) 是全球最先进的企业级微服务持久化分布式工作流编排引擎。它采用**事件溯源（Event Sourcing）与虚拟内存重放（Virtual Memory Replay）**技术，使得开发者可以使用普通的高级编程语言（Go / TypeScript / Java / Python）编写任意长寿命的工作流（甚至持续运行数年），即使底层机器断电、Kubernetes Pod 重启，工作流代码的执行进度与局部变量也能被完全恢复。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Temporal Client SDK App                         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ gRPC 双向流
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Temporal Cluster Core (Go)                          │
├───────────────────────────────┬────────────────────────────────────────┤
│ Frontend Gateway Service      │ History Service & Event Engine         │
│ - gRPC 限流与鉴权校验         │ - 状态机持久化事件流 (Events List)     │
│ - 跨集群多租户路由转发       │ - 确定性执行重放 (Deterministic Replay) │
├───────────────────────────────┼────────────────────────────────────────┤
│ Matching Service              │ Timer & Internal Queue Manager         │
│ - 任务队列 (TaskQueue) 分发   │ - 分布式定时器与延时触发调度           │
│ - Worker 长轮询拉取匹配       │ - 信号 (Signal) 与查询 (Query) 协调器  │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Temporal 为分布式系统测试树立了最高水准的自动化验证范式：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Workflow Determinism Replay**| 校验代码逻辑变更是否破坏历史事件树兼容性 | Temporal SDK Replayer | 纯离线加载历史 JSON 事件流，无需集群 |
| **TestWorkflowEnvironment** | 虚拟时钟快进、模拟几天后的定时器触发、Activity 故障注入 | SDK 测试套件（自带内嵌 Server） | 内存单进程虚拟集群，毫秒级模拟长时间流逝 |
| **Activity Retry & Backoff** | 网络超时、异常抛出、指数退避间隔校验 | SDK Test Framework | 拦截 Activity 调用次数与时间戳 |
| **Signals & Queries Tests** | 外部异步事件触发状态跃迁、只读查询返回当前内存变量 | SDK Test Framework | 驱动式断言（Step-by-step） |
| **Dynamic Versioning Tests** | 使用 `workflow.GetVersion()` 升级工作流代码且不影响旧实例 | Replay Test Suite | 双版本历史记录交替执行对比 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：Workflow 确定性执行检验与事件溯源重放（Replayer）测试

#### 3.1.1 业务功能与测试目标
Temporal 工作流代码必须是**严格确定性（Deterministic）**的，禁止在工作流函数内部直接读取系统时间 `time.Now()`、生成随机数或发起原生 HTTP 请求。若开发者在迭代中不小心引入了破坏确定性的改动，历史工作流实例在重放时会发生 `Non-Deterministic Error`。测试必须拦截此类致命代码退化。

#### 3.1.2 测试用例实现（Go SDK 测试）
```go
package workflow_test

import (
	"testing"

	"github.com/stretchr/testify/require"
	"go.temporal.io/sdk/worker"
)

func TestWorkflow_ReplayHistoryWithoutNonDeterministicError(t *testing.T) {
	// 1. 初始化重放器，注册当前最新版本的工作流代码
	replayer := worker.NewWorkflowReplayer()
	replayer.RegisterWorkflow(MyOrderFulfillmentWorkflow)

	// 2. 加载生产环境中真实运行完成并归档的历史事件 JSON 文件 (包含数十个 Activity 交互历史)
	historyJsonFile := "testdata/order_workflow_v1_history.json"

	// 3. 执行重放断言：回放全部事件，验证决策树完全吻合
	err := replayer.ReplayWorkflowHistoryFromJSONFile(nil, historyJsonFile)
	require.NoError(t, err, "Workflow history replay failed! Detected non-deterministic change.")
}
```

---

### 3.2 模块二：虚拟时钟快进（Time Skipping）与定时器唤醒测试

#### 3.2.1 业务功能与测试目标
业务场景：用户注册后，系统等待 7 天（`workflow.Sleep(ctx, 7*24*time.Hour)`），若未完成首单则发送唤醒邮件。
自动化测试不可能真正等待 7 天。测试环境必须能**自动将虚拟时钟跳跃 7 天**，瞬间唤醒定时器并断言后续行为。

#### 3.2.2 测试用例实现（Go SDK 测试）
```go
package workflow_test

import (
	"testing"
	"time"

	"github.com/stretchr/testify/mock"
	"github.com/stretchr/testify/require"
	"go.temporal.io/sdk/testsuite"
)

func TestWorkflow_Sleep7DaysWithTimeSkipping(t *testing.T) {
	testSuite := &testsuite.WorkflowTestSuite{}
	env := testSuite.NewTestWorkflowEnvironment()

	// 1. Mock 唤醒邮件发送的 Activity
	env.OnActivity("SendReminderEmail", mock.Anything, "user_123").Return(nil).Once()

	// 2. 异步启动长达 7 天等待的工作流
	env.ExecuteWorkflow(OnboardingWorkflow, "user_123")

	// 3. 断言执行瞬间完成（虚拟时钟瞬间推进 7 天，物理执行耗时低于 100 毫秒）
	require.True(t, env.IsWorkflowCompleted())
	require.NoError(t, env.GetWorkflowError())
	env.AssertExpectations(t)
}
```

---

### 3.3 模块三：Activity 异常重试策略（RetryPolicy）与指数退避（Exponential Backoff）测试

#### 3.3.1 业务功能与测试目标
定义 Activity 重试策略：初始间隔 1 秒，Backoff 系数 2.0，最大尝试 4 次。测试必须验证：
1. 连续失败时，每次重试的虚拟时钟间隔严格按照 1s -> 2s -> 4s 翻倍。
2. 超过最大尝试次数后，精准向上层工作流抛出 `ActivityFailure`。

#### 3.3.2 测试用例实现
```go
func TestActivity_ExponentialBackoffRetryPolicy(t *testing.T) {
	testSuite := &testsuite.WorkflowTestSuite{}
	env := testSuite.NewTestWorkflowEnvironment()

	var attemptTimes []time.Time

	// 模拟一个前 3 次均因网络中断抛错的 Activity
	env.OnActivity("CallPaymentGateway", mock.Anything, mock.Anything).
		Return(func(ctx context.Context, amount int) error {
			attemptTimes = append(attemptTimes, env.Now())
			return errors.New("network connection reset")
		}).Times(3)

	env.ExecuteWorkflow(PaymentWorkflow, 100)

	require.Error(t, env.GetWorkflowError())
	// 验证触发了 3 次调用
	require.Len(t, attemptTimes, 3)

	// 验证指数退避间隔递增
	interval1 := attemptTimes[1].Sub(attemptTimes[0])
	interval2 := attemptTimes[2].Sub(attemptTimes[1])
	require.Equal(t, 1*time.Second, interval1)
	require.Equal(t, 2*time.Second, interval2)
}
```

---

### 3.4 模块四：异步信号（Signal）通道交互与只读查询（Query）状态测试

#### 3.4.1 业务功能与测试目标
工作流运行在等待状态时，测试可以随时向工作流发送 Signal 改变其内部状态，并调用 Query 获取当前计算指标，而不需要将工作流提前终止。

#### 3.4.2 测试用例实现
```go
func TestWorkflow_SignalAndQueryInteraction(t *testing.T) {
	testSuite := &testsuite.WorkflowTestSuite{}
	env := testSuite.NewTestWorkflowEnvironment()

	// 注册执行延时工作流
	env.RegisterDelayedCallback(func() {
		// 1. 发起 Query：此时计数应为初始值 0
		val, err := env.QueryWorkflow("getCurrentCount")
		require.NoError(t, err)
		var count int
		val.Get(&count)
		require.Equal(t, 0, count)

		// 2. 发送异步 Signal：追加数值 42
		env.SignalWorkflow("incrementSignal", 42)
	}, 1*time.Minute)

	env.ExecuteWorkflow(CounterWorkflow)

	// 3. 校验最终完成结果接收到了 Signal
	var finalResult int
	env.GetWorkflowResult(&finalResult)
	require.Equal(t, 42, finalResult)
}
```

---

## 4. 测试工程化与基础设施

1. **TestWorkflowEnvironment 纯内存沙箱**：
   * Temporal 官方测试库提供完整的虚拟集群运行时，在单个测试进程中完成 Matching、History 和 Timer 的极速交互，没有任何 Docker 或外置数据库开销。
2. **生产环境事件历史脱敏 Golden 库**：
   * 在代码仓库维护典型复杂业务的历史 JSON 事件流，升级任何公共工作流代码时必须通过全量 Replay 扫描，杜绝线上发生非确定性锁死。

---

## 5. 对同类复杂系统的参考价值

* **虚拟时钟驱动长时间跨度自动化测试**：对于包含“延时重试、多天后提醒、跨月结账”的系统，切忌在代码中硬编码真实时间，通过将时间虚拟化，可在百毫秒内完成横跨数年的业务闭环断言。
