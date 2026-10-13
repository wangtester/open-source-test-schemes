# CrewAI 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[CrewAI](https://github.com/crewAIInc/crewAI) 是全球领先的企业级多智能体协同（Multi-Agent Collaboration）与角色分工工作流自动化框架。系统基于 Python 构建，创新定义了具有明确角色（Role）、目标（Goal）与背景故事（Backstory）的 Agent 团队（Crew），支持顺序执行（Sequential）、层次化管理执行（Hierarchical with Manager）与智能体间的**相互委派（Task Delegation）**。

### 1.2 被测系统核心组成
1. **多智能体编排调度器（Crew & Process Engine）**：管理任务（Task）依赖图，控制数据沿流水线向下游 Agent 传递。
2. **智能体间任务委派机制（Agent Delegation Hub）**：支持 Agent A 在思考中主动调用系统工具，将子任务分配给更擅长的 Agent B 并等待回执。
3. **层次化管理者大脑（Hierarchical Manager LLM）**：作为团队管理者动态评估任务完成度并实时重调资源。
4. **工具沙箱与重试控制器（Tool Execution & Retry Controller）**：保障外部 Tool 调用在失败时触发智能重试。

### 1.3 自动化测试的主要难点与挑战
* **智能体间相互委派导致的死锁与无限循环**：Agent A 委派给 Agent B，Agent B 又将任务委派回 Agent A。
* **多智能体上下文膨胀与 Token 溢出**：多人交互时共享记忆的增量裁剪断言。
* **层次化管理模式下动态任务拆分的非确定性**：Manager 决策路径的弹性校验。

---

## 2. 测试架构全景与分层策略

CrewAI 建立了以死锁防护与多角色契约交互为核心的自动化测试体系：

```
┌────────────────────────────────────────────────────────┐
│             Crew Sequential & Hierarchical E2E Suite   │  编排全链路：验证从输入需求到最终团队输出的多 Agent 流转
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Agent Task Delegation & Deadlock Guard Tests      │  委派协同层：测试智能体间互相分配任务与死锁熔断
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Tool Execution Failure & Self-Correction Tests │  工具容错层：模拟 Tool 抛异常后 Agent 的参数调整行为
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Agent Role, Memory & Task Unit Tests           │  实体单元层：验证 Agent 属性绑定、短期记忆写入与格式化
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：智能体相互委派死锁防护（Deadlock & Loop Guard）测试

#### 3.1.1 业务功能背景
研究员 Agent 决定将一个分析任务委派给作家 Agent，作家 Agent 试图再次将任务反向委派给研究员 Agent。系统必须识别出循环依赖或超过最大委派深度（`max_delegations`），强行禁止二次委派并促使当前 Agent 自行完成。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 委派机制测试）**：
```python
import pytest
from crewai import Agent, Task, Crew, Process

def test_delegation_loop_is_prevented_by_guard(mock_llm):
    # 1. 构造两个具有委派权限的互锁 Agent
    agent_a = Agent(role="Researcher", goal="Research topics", backstory="Expert", llm=mock_llm, allow_delegation=True)
    agent_b = Agent(role="Writer", goal="Write drafts", backstory="Author", llm=mock_llm, allow_delegation=True)

    # 2. 模拟双方均输出向对方委派工具调用的死循环指令
    mock_llm.set_infinite_delegation_loop_responses()

    task = Task(description="Produce final report", expected_output="Report", agent=agent_a)
    crew = Crew(agents=[agent_a, agent_b], tasks=[task], max_rpm=100)

    # 3. 触发团队执行并断言安全退出
    result = crew.kickoff()

    # 4. 严格硬断言：系统未发生死锁挂死，在达到最大委派深度时强制中断并返回兜底
    assert result is not None
    assert agent_a.delegation_count <= crew.max_delegation_limit
```

---

### 3.2 功能二：层次化团队（Hierarchical Process）任务依赖下发测试

#### 3.2.1 业务功能背景
配置 `process=Process.hierarchical`。系统自动挂载 Manager LLM，当上游“数据收集”任务未产出预期输出前，Manager 绝对不能提前调度下游“格式化报告”任务。

#### 3.2.2 测试用例设计方案
* **测试编排**：
  1. 编排包含依赖顺序的 Task 1 与 Task 2。
  2. 捕获每个 Agent 的执行时间戳与输入上下文。
  3. **断言**：Task 2 的执行起始时间严格滞后于 Task 1 的完成时间，且 Task 1 的输出被完整作为上下文参数注入 Task 2。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **多 Agent 状态机步数预算熔断**：
   * 所有测试用例强制设置超时装饰器 `@pytest.mark.timeout(10)`，杜绝多 Agent 协同假死拖垮整个流水线。
2. **纯内存 Mock 记忆与上下文收集器**：
   * 测试套件将 Chroma 向量长程记忆替换为内存简单列表，保证多 Agent 记忆读写在单机单线程极速回归。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **多 Agent 系统测试必须重点设计“死锁与循环委派检测”**：单一 Agent 只会死循环，多 Agent 会产生复杂的网状死锁；在测试中故意让 Agent 之间互相踢皮球，是验证协同调度引擎健壮度的终极考卷。
* **严格隔离任务状态与时序拓扑**：在编排多角色任务时，必须通过断言调用时间戳与依赖产物的传递链，验证流程严格受控。
