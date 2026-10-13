# OpenHands 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[OpenHands](https://github.com/All-Hands-AI/OpenHands)（原 OpenDevin）是开源自主 AI 软件工程师平台。系统支持通过自然语言需求描述，自主进行代码库探索、代码修改、环境排查、测试运行及 Git 提交。

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Web Studio (React + Monaco + xterm.js)               │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ WebSocket / SSE 双向事件流
┌───────────────────────────────────▼────────────────────────────────────┐
│                    EventStream (统一异步事件总线)                       │
│      Action (用户/Agent 决策行为)  ◄──►  Observation (执行环境观测反馈)    │
└───────────────┬────────────────────────────────────────┬───────────────┘
                │                                        │
┌───────────────▼──────────────┐         ┌───────────────▼───────────────┐
│ Agent Controller (状态机内核) │         │ Runtime Sandbox (Docker 沙箱)  │
│ - CodeActAgent / Planner     │         │ - Bash 命令执行环境 (execa)    │
│ - LLM Provider 适配与容错    │         │ - 虚拟/真实文件系统读写        │
│ - 步数与成本硬熔断           │         │ - Headless 浏览器交互环境     │
└──────────────────────────────┘         └───────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

OpenHands 的用例套件完整覆盖了从纯单元算法到全隔离 Docker 物理沙箱的各层级：

| 测试层级 | 被测目标与用例范围 | 运行环境与技术选型 | 隔离与清理机制 |
| :--- | :--- | :--- | :--- |
| **Unit Tests** | Action/Observation 序列化、AST 解析、Git 补丁提取、LLM 提示词模板 | Pytest / pytest-mock | 无需外部依赖，毫秒级执行 |
| **Agent Controller Tests** | 智能体状态机流转、死循环熔断、LLM 异常重试、多轮对话记忆压缩 | Mock LLM + 内存 EventStream | 隔离外部 LLM API，纯内存运行 |
| **Runtime Sandbox Tests** | Docker 容器生成、Bash 执行、端口映射、文件修改物理落盘、权限隔离 | 真实 Docker Daemon (`pytest-asyncio`) | Fixture 级别容器生命周期管控与强制销毁 |
| **Tools & Browser Tests** | 浏览器交互工具（Playwright in Docker）、Jupyter 交互、Web 搜索工具 | 无头 Chromium + Docker 沙箱 | 独立沙箱与端口隔离 |
| **Web Frontend E2E** | React 交互界面、Monaco Diff 渲染、xterm 终端交互、对话树分支切换 | Playwright (Node.js/TypeScript) | Mock 后端 WebSocket 或轻量受控容器 |
| **Evaluation Benchmark** | Agent 解决真实 GitHub 问题的端到端成功率判定 | Docker-in-Docker + 真实构建测试 | 严格只读挂载基准仓库，隔离网络 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：EventStream 异步事件总线契约测试

#### 3.1.1 业务功能与测试目标
EventStream 是整个 OpenHands 的中枢神经，所有用户输入、Agent 决策（`Action`）与系统响应（`Observation`）均转换为强类型的事件。测试必须保证：
1. Action 与 Observation 的 JSON 双向序列化契约完全符合 Pydantic 模型。
2. 异步订阅者（Subscribers）能够按严格的时间戳顺序接收事件，且单订阅者异常不阻塞总线。

#### 3.1.2 测试用例实现
```python
import pytest
import asyncio
from openhands.events.stream import EventStream
from openhands.events.action import CmdRunAction, FileEditAction
from openhands.events.observation import CmdOutputObservation, ErrorObservation
from openhands.events.serialization import event_to_dict, event_from_dict

@pytest.mark.asyncio
async def test_event_stream_serialization_roundtrip_and_order():
    stream = EventStream(sid="test_session_001")
    received_events = []

    async def subscriber(event):
        received_events.append(event)

    stream.subscribe(subscriber)

    # 1. 模拟派发命令执行 Action
    action = CmdRunAction(command="npm run build", keep_prompt=False)
    serialized = event_to_dict(action)
    assert serialized["action"] == "run"
    assert serialized["args"]["command"] == "npm run build"

    deserialized = event_from_dict(serialized)
    assert isinstance(deserialized, CmdRunAction)
    assert deserialized.command == "npm run build"

    # 2. 派发并验证总线有序投递
    await stream.add_event(action)
    obs = CmdOutputObservation(content="Build successful", command_id=action.id, exit_code=0)
    await stream.add_event(obs)

    # 3. 验证接收顺序与数据一致性
    assert len(received_events) == 2
    assert isinstance(received_events[0], CmdRunAction)
    assert isinstance(received_events[1], CmdOutputObservation)
    assert received_events[1].exit_code == 0
```

---

### 3.2 模块二：Runtime Docker 沙箱环境物理生命周期与命令执行测试

#### 3.2.1 业务功能与测试目标
Runtime 负责动态启动 Docker 容器，并执行高风险的 Shell 命令。测试必须覆盖：
1. 容器正常拉起、环境健康检测与彻底销毁（清理残留）。
2. 执行 Shell 产生标准输出、标准错误、退出码的精确捕获。
3. 超时命令的主动 SIGKILL 信号终止能力，防止僵尸进程。

#### 3.2.2 测试用例实现
```python
import pytest
import pytest_asyncio
from openhands.runtime import DockerRuntime
from openhands.events.action import CmdRunAction

@pytest_asyncio.fixture
async def docker_runtime():
    runtime = DockerRuntime(
        image_name="openhands-runtime-testing:latest",
        container_name_prefix="test_openhands_runner",
        timeout=30
    )
    await runtime.init()
    yield runtime
    # 强制清理：测试结束必须确保容器被彻底 stop 和 rm
    await runtime.close()

@pytest.mark.asyncio
async def test_runtime_command_execution_and_timeout(docker_runtime):
    # 1. 正常命令与环境变量断言
    cmd = CmdRunAction(command="echo 'TEST_VAR=123' && uname -s")
    obs = await docker_runtime.run_action(cmd)
    
    assert obs.exit_code == 0
    assert "TEST_VAR=123" in obs.content
    assert "Linux" in obs.content

    # 2. 超时命令硬终止断言（模拟 Agent 发起 sleep 1000）
    timeout_cmd = CmdRunAction(command="sleep 60", timeout=2)
    timeout_obs = await docker_runtime.run_action(timeout_cmd)

    # 退出码判定与超时报错断言
    assert timeout_obs.exit_code != 0
    assert "Command timed out" in timeout_obs.content or timeout_obs.exit_code == 137
```

---

### 3.3 模块三：文件系统读写、AST 级代码编辑与 Git Diff 补丁测试

#### 3.3.1 业务功能与测试目标
Agent 修改代码不能盲目重写整个文件，通常采用针对特定行范围的补丁修改（Aider 模式）或全量替换。测试必须覆盖：
1. 多行代码就地替换与缩进保持。
2. 语法树（AST）校验：防止 Agent 破坏语法生成非法的代码。
3. 试图访问容器外部路径（Path Traversal）的安全拒绝。

#### 3.3.2 测试用例实现
```python
import pytest
from openhands.runtime import DockerRuntime
from openhands.events.action import FileWriteAction, FileEditAction, FileReadAction

@pytest.mark.asyncio
async def test_file_editing_and_security_traversal(docker_runtime):
    # 1. 初始文件写入
    initial_code = "def add(a, b):\n    # TODO: implement\n    pass\n"
    write_act = FileWriteAction(path="/workspace/math_utils.py", content=initial_code)
    write_obs = await docker_runtime.run_action(write_act)
    assert write_obs.is_success()

    # 2. 使用 Patch / Edit 进行局部精准替换
    edit_act = FileEditAction(
        path="/workspace/math_utils.py",
        old_content="    # TODO: implement\n    pass",
        new_content="    return a + b"
    )
    edit_obs = await docker_runtime.run_action(edit_act)
    assert edit_obs.is_success()

    # 3. 物理读回校验完整性
    read_act = FileReadAction(path="/workspace/math_utils.py")
    read_obs = await docker_runtime.run_action(read_act)
    assert "return a + b" in read_obs.content
    assert "pass" not in read_obs.content

    # 4. 安全攻击路径测试：禁止遍历容器外敏感文件
    hack_act = FileReadAction(path="/etc/shadow")
    hack_obs = await docker_runtime.run_action(hack_act)
    assert not hack_obs.is_success()
    assert "Permission denied" in hack_obs.content or "Access restricted" in hack_obs.content
```

---

### 3.4 模块四：AgentController 状态机控制与死循环熔断测试

#### 3.4.1 业务功能与测试目标
AgentController 调度 Agent 的决策循环。测试必须覆盖：
1. 状态跃迁：`INIT` -> `RUNNING` -> `PAUSED` / `FINISHED` / `ERROR`。
2. 步数硬熔断：当 Agent 超过设定的 `max_iterations` 时，系统必须立即安全熔断，不能继续消耗 Token。
3. Agent 遇到错误命令时的自我修复重试机制。

#### 3.4.2 测试用例实现
```python
import pytest
from unittest.mock import AsyncMock, MagicMock
from openhands.controller.agent_controller import AgentController
from openhands.controller.state.state import AgentState
from openhands.events.stream import EventStream
from openhands.events.action import MessageAction, CmdRunAction
from openhands.events.observation import CmdOutputObservation

@pytest.mark.asyncio
async def test_agent_controller_hits_max_iterations_and_pauses():
    event_stream = EventStream("test_controller_session")
    
    # 构造虚假智能体：始终输出无效的尝试指令
    mock_agent = MagicMock()
    mock_agent.step = AsyncMock(return_value=CmdRunAction(command="echo 'retry'"))
    
    controller = AgentController(
        agent=mock_agent,
        event_stream=event_stream,
        max_iterations=3,
        max_budget_per_task=1.0
    )

    # 触发任务启动
    await controller.start_task("Solve this recurring bug")
    
    # 模拟沙箱环境对命令的自动反馈
    for _ in range(3):
        await event_stream.add_event(CmdOutputObservation(content="retry", exit_code=1))
        await controller.step()

    # 验证达到步数上限后的状态断言
    assert controller.state == AgentState.PAUSED
    assert controller.step_count == 3
    assert controller.hit_max_iterations is True
```

---

### 3.5 模块五：Headless 浏览器工具（BrowserTool）交互测试

#### 3.5.1 业务功能与测试目标
OpenHands 配备浏览器交互能力（用于查看部署后的 Web 页面或文档）。测试必须保证：
1. 能够向受测网页发起跳转并截取页面视口快照。
2. 元素定位、点击与表单提交操作在 Docker 内部能够无报错执行。

#### 3.5.2 测试用例实现
```python
import pytest
from openhands.events.action import BrowseURLAction
from openhands.events.observation import BrowserOutputObservation

@pytest.mark.asyncio
async def test_browser_tool_navigation_and_dom_snapshot(docker_runtime):
    # 派发访问本地受控 HTML 服务的 Action
    browse_act = BrowseURLAction(url="https://example.com")
    obs = await docker_runtime.run_action(browse_act)

    assert isinstance(obs, BrowserOutputObservation)
    assert obs.is_success()
    # 验证返回了 DOM 树结构与截图 Base64
    assert "Example Domain" in obs.content
    assert obs.screenshot is not None
    assert len(obs.screenshot) > 100
```

---

### 3.6 模块六：Web 协同工作台端到端（E2E）测试

#### 3.6.1 业务功能与测试目标
基于 Playwright 驱动前端浏览器，验证：
1. 用户在输入框键入需求后，对话树即时呈现用户气泡。
2. 后端推送文件变更事件时，Monaco Editor 自动切入 Diff 视图。
3. 终端窗口正确显示 Bash 交互输出。

#### 3.6.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('OpenHands Studio E2E Suite', () => {
  test('User sends task, Monaco shows diff and terminal shows build', async ({ page }) => {
    await page.goto('http://localhost:3000');

    // 1. 验证工作区与文件树加载完成
    await expect(page.locator('[data-testid="file-tree"]')).toBeVisible();

    // 2. 输入任务需求
    const inputArea = page.locator('textarea[placeholder*="Ask OpenHands"]');
    await inputArea.fill('Create a fibonacci function in math.py and run tests');
    await page.keyboard.press('Enter');

    // 3. 断言智能体响应状态气泡
    await expect(page.locator('[data-testid="agent-message-bubble"]')).toContainText('Analyzing task', { timeout: 10000 });

    // 4. 断言 Monaco Editor 收到并渲染 Diff 对比视图
    const monacoDiff = page.locator('.monaco-diff-editor');
    await expect(monacoDiff).toBeVisible({ timeout: 15000 });
    await expect(monacoDiff).toContainText('def fibonacci(n):');

    // 5. 校验终端选项卡收到并展示 pytest 运行日志
    await page.click('button:has-text("Terminal")');
    const terminalOutput = page.locator('.xterm-rows');
    await expect(terminalOutput).toContainText('passed in', { timeout: 20000 });
  });
});
```

---

## 4. 测试工程化与持续集成（CI）保障

1. **Docker-in-Docker (DinD) 并发保护**：
   * GitHub Actions 采用宿主机 Docker Socket 挂载，每个并发 Job 分配唯一的 `CONTAINER_NAME_PREFIX` 与虚拟子网，杜绝端口冲突。
2. **LLM 录制与回放（VCR 模式）**：
   * 对涉及真实提示词与状态机的测试，采用网络 Mock 拦截，将 LLM 交互录制为 YAML 文件，避免 CI 流程因网络抖动或模型版本漂移失败。

---

## 5. 对同类复杂系统的参考价值

* **事件溯源（Event Sourcing）的自动化测试**：将复杂系统交互抽象为一维有序的事件流，使得测试只需要断言事件序列（Action -> Observation），极大降低了异步系统的断言复杂度。
* **物理沙箱的绝对隔离与防御性销毁**：自动化测试必须使用 `try...finally` 与测试夹具强制清理外部进程与容器，杜绝任何中间状态污染。
