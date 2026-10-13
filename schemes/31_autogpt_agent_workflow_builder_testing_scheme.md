# AutoGPT 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[AutoGPT](https://github.com/Significant-Gravitas/AutoGPT) 是全球知名的开源自主智能体（Autonomous Agent）平台与多 Agent 协同工作流系统。项目在迭代过程中大量依托自动化 AI 编程流水线构建。系统包含基于 Python 的核心智能体协议规范（Forge / Agent Protocol），以及基于现代 Web 的可视化工作流编排画布（Workflow Builder）与人机交互控制台。

### 1.2 被测系统核心组成
1. **Agent 核心执行循环（Agent Execution Loop & Memory）**：多轮思考、任务分解、情境记忆检索与工具行动派发。
2. **人机协同审批拦截网关（Human-in-the-Loop Approval Gateway）**：对写文件、Shell 执行、外部网络请求等敏感/高危行动执行状态挂起，等待用户授权。
3. **工作流图拓扑执行器（Workflow Graph Runner）**：支持将多个专门 Agent（如研究员、程序员、审查员）编排为协作拓扑网格。
4. **Agent Protocol 统一协议接口**：遵循社区开放标准暴露 Task、Step、Artifact RESTful 接口。

### 1.3 自动化测试的主要难点与挑战
* **长程自主决策的不可预测性**：Agent 规划的步骤路径可能各异，如何设计不被特定推理路径绑架的弹性断言。
* **人机协同审批挂起状态的异步等待**：状态机进入长轮询等待时，测试程序需模拟用户在前端点击“同意”或“拒绝”并验证双向流转。
* **多智能体消息传递的死锁与竞态**：两个 Agent 相互等待对方回执导致测试超时的排查与治理。

---

## 2. 测试架构全景与分层策略

AutoGPT 演进出了基于 Agent Protocol 规范的标准化测试防线：

```
┌────────────────────────────────────────────────────────┐
│             Web UI & Workflow Builder E2E              │  前端交互层：驱动节点编排、审批气泡点击、运行看板
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│         Agent Protocol Compliance Integration Suite    │  规范契约层：测试 Task/Step REST API 是否符合社区标准
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Human-in-the-loop Approval & Tool Dispatch Tests  │  安全网关层：模拟高危命令触发挂起、批准与拒绝后的自愈
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Agent Loop & Memory Unit Tests (Pytest)        │  决策计算层：纯算法验证任务分解、短期与情境记忆打捞
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：人机协同高危操作审批拦截（Human-in-the-Loop Gateway）

#### 3.1.1 业务功能背景
当 Agent 决定执行一个涉及敏感文件删除或高权限命令（如 `bash -c "rm -rf build"`）时，系统必须自动拦截执行，将当前 Step 状态置为 `WAITING_USER_INPUT`。用户在 Web 页面点击“拒绝并留言：请先备份”，Agent 必须感知到拒绝理由，调整策略先执行备份。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 状态机异步集成测试）**：
```python
import pytest
from autogpt.core.runner import AgentRunner
from autogpt.core.status import StepStatus

@pytest.mark.asyncio
async def test_high_risk_action_triggers_approval_interception():
    runner = AgentRunner(task="Clean up old build artifacts")
    
    # 1. 推进第一步：Agent 产生删除动作
    step_result = await runner.step()
    
    # 2. 断言高危操作被网关自动挂起，未直接在物理环境执行
    assert step_result.status == StepStatus.WAITING_USER_INPUT
    assert step_result.action_to_approve.name == "execute_shell"
    assert "rm -rf" in step_result.action_to_approve.params["command"]

    # 3. 模拟用户通过 Web 接口下发拒绝并附加反馈指令
    next_step = await runner.submit_user_feedback(
        approved=False,
        feedback="Do not delete directly. Create a backup zip first."
    )

    # 4. 断言 Agent 状态机吸收了人类反馈，修正了下一步行动
    assert next_step.status == StepStatus.RUNNING
    assert next_step.current_action.name == "execute_shell"
    assert "zip" in next_step.current_action.params["command"]
```

---

### 3.2 功能二：Agent Protocol 标准规范一致性与状态断言

#### 3.2.1 业务功能背景
AutoGPT 作为后端服务，必须严格实现开源 Agent Protocol 规范：`POST /ap/v1/agent/tasks` 创建任务，`POST /ap/v1/agent/tasks/{task_id}/steps` 逐步执行，并能下载生成的 Artifact 文件。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
  * 使用标准的开源协议测试套件向 AutoGPT 发送一组基准 API 请求。
  * 断言返回的每个 JSON 对象的 Schema 严格满足 OpenAPI 规范。
  * 断言任务完成后，`GET /ap/v1/agent/tasks/{task_id}/artifacts` 能列出并下载生成的所有文件。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **确定性 Mock LLM 响应录像机**：
   * 采用预定义的 JSON 剧本（Scripted Scenario），模拟多步复杂规划回复，测试期间完全断开真实大模型网络连接。
2. **多 Agent 协同超时死锁守护器**：
   * 在 Pytest 中为每个多 Agent 交互用例配置 15 秒超时预算，若两个 Agent 陷入相互等待则立刻中断并打印双方的未完成事件队列。

---

## 5. 对复杂系统的实战借鉴意义

* **高自主权系统必须把“审批与拦截机制”作为最高优先级测试**：对于 Agent 或自动化流程平台，测试的重心不仅是“能否自主完成”，更在于“遇到危险时能否绝对停下来等候指令”。
* **遵从行业标准契约（Agent Protocol）简化测试架构**：对复杂系统抽象出标准化的协议契约层，可以让通用协议测试套件直接复用，极大提升系统的通用性与测试严谨度。
