# OpenHands 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[OpenHands](https://github.com/All-Hands-AI/OpenHands)（原 OpenDevin）是全球领先的开源自主 AI 软件工程师平台。系统在研发过程中深度融合了 AI 编码智能体的自举开发与社区协同。前端采用 React + TypeScript + Monaco Editor + xterm.js 终端模拟器，后端采用 Python (FastAPI) 驱动，结合 Docker 容器沙箱实现安全的代码隔离执行。

### 1.2 被测系统核心组成
1. **EventStream 异步事件总线**：连接前端用户、Agent 决策内核与沙箱环境的统一事件管道。
2. **Docker 运行时与代码沙箱（Runtime Sandbox）**：提供安全的 Linux 容器环境，负责执行 Shell 命令、文件编辑与浏览器无头操作。
3. **Agent 执行循环（Agent Controller & State Machine）**：驱动 Agent 分析需求、生成行动计划、调用工具并观测执行结果。
4. **Web 协同工作台**：集成多窗口分栏（对话树、Monaco 代码 Diff 对比器、动态文件树、实时交互终端与浏览器预览窗口）。

### 1.3 自动化测试的主要难点与挑战
* **Agent 陷入死循环或僵尸操作**：Agent 在执行复杂编译报错时可能反复尝试相同命令，测试必须具备超步数硬熔断断言。
* **Docker 容器生命周期与资源泄露**：每个测试用例启动独立沙箱容易造成测试机 Docker 容器堆积与网络端口占满。
* **异步事件总线的时序乱序**：Action 事件与 Observation 事件在 WebSocket 传输中的异步配对校验。

---

## 2. 测试架构全景与分层策略

OpenHands 建立了严格的隔离沙箱与事件流驱动自动化测试体系：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Full-Stack Studio E2E           │  端到端层：驱动 Web 界面、实时终端输入与 Monaco Diff
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Docker Runtime & Sandbox Isolation Tests          │  沙箱环境层：真实创建临时容器，验证 Bash 命令执行与隔离
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Agent Controller & EventStream Integration     │  智能体层：使用 Mock LLM 回归状态机流转与决策断言
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Tools, Code Edit & AST Unit Tests              │  工具单元层：验证文件补丁应用（Aider/Patch）、正则与语法
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：Docker 沙箱命令执行与文件副作用物理验证

#### 3.1.1 业务功能背景
Agent 发出 `CmdRunAction(command="pytest")` 或写入代码补丁，沙箱容器必须在受限环境下执行，捕获命令输出（Observation），并在磁盘上产生实际文件变动。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 沙箱集成测试）**：
```python
import pytest
from openhands.runtime import DockerRuntime
from openhands.events.action import CmdRunAction, FileWriteAction

@pytest.mark.asyncio
async def test_docker_runtime_file_write_and_bash_execution():
    # 1. 启动临时受测容器沙箱
    runtime = DockerRuntime(sandbox_image="openhands-runtime:latest")
    await runtime.init()

    try:
        # 2. 派发写入文件 Action
        write_action = FileWriteAction(path="/workspace/main.py", content="print('Hello OpenHands')")
        obs = await runtime.run_action(write_action)
        assert obs.is_success()

        # 3. 派发运行 Bash 命令 Action
        cmd_action = CmdRunAction(command="python /workspace/main.py")
        cmd_obs = await runtime.run_action(cmd_action)
        
        # 4. 物理副作用断言：退出码必须为 0，标准输出精准捕获
        assert cmd_obs.exit_code == 0
        assert "Hello OpenHands" in cmd_obs.content
    finally:
        # 5. 必须彻底销毁容器，防止资源泄漏
        await runtime.close()
```

---

### 3.2 功能二：Agent 决策状态机与步数硬熔断保护

#### 3.2.1 业务功能背景
当 Agent 面对一个反复报错的任务时，如果连续达到最大迭代步数（如 10 步），系统必须自动终止执行，更新状态为 `PAUSED` 或 `ERROR`，并在 EventStream 中广播停止通知，杜绝无限死循环。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
@pytest.mark.asyncio
async def test_agent_controller_hits_max_iterations():
    event_stream = EventStream()
    mock_llm = AlwaysFailingLLM() # 故意总是返回相同无用建议的模型桩
    controller = AgentController(
        agent=CodeAgent(llm=mock_llm),
        event_stream=event_stream,
        max_iterations=5,
    )

    # 启动控制器执行循环
    await controller.start_task("Fix the broken build")
    await controller.run_until_done()

    # 严格断言：迭代计数严格等于 5，且状态正确翻转为停止
    assert controller.step_count == 5
    assert controller.state == AgentState.PAUSED
    assert any("Maximum iterations reached" in str(e) for e in event_stream.get_events())
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Docker-in-Docker CI Runner 配置**：
   * 在 GitHub Actions 启用宿主机 Docker Socket 挂载，利用专用轻量镜像跑真实子容器测试。
2. **EventStream 异步测试辅助器（Event Waiter）**：
   * 封装 `event_stream.wait_for_event(cls, timeout=5)`，避免写恶意的 `time.sleep`，实现基于异步事件流的确定性唤醒。

---

## 5. 对复杂系统的实战借鉴意义

* **将 Agent 架构解耦为 Action/Observation 事件双工管道**：对于任何 AI 驱动系统，所有指令与回执标准化为事件，不仅能让 UI 与后端完全解耦，更能让自动化测试以极低成本拦截和断言每一步骤。
* **沙箱环境必须有强制自动回收器（Auto-teardown）**：在集成测试中动态拉起独立执行容器时，使用 Python `try...finally` 或 `pytest fixture` 确保容器句柄必被销毁，是测试流水线健康运行的基石。
