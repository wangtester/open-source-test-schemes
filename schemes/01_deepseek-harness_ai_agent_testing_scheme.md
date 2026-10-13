# DeepSeek Harness 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 是针对复杂多步骤 AI Agent 研发与评估的高性能线束评测系统。该系统通过提供受控执行容器、确定性网络模拟与跨会话状态快照，评估大语言模型在工具调用、长程代码推理与复杂环境下的决策稳健性。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DeepSeek Harness Runner CLI                     │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ 控制调度与评估指标聚合
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Harness Runtime Core (Python/Rust)                  │
├───────────────────────────────┬────────────────────────────────────────┤
│ AgentLoop 调度与时序控制      │ Network WireMock 故障注入总线          │
│ - 工具调用有序认领与执行      │ - 24 种传输层/协议层异常拦截注入        │
│ - 超步数/超时硬熔断状态机     │ - 网络抖动、丢包与 SSE 流断流重连       │
├───────────────────────────────┼────────────────────────────────────────┤
│ Subprocess / Docker Sandbox   │ Session Snapshot & Rollback Engine     │
│ - 子进程树追踪与安全 cgroups  │ - v1 ~ v3 跨版本会话状态双向迁移       │
│ - 资源上限管控与自动清理      │ - 内存状态差分比对与确定性恢复         │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

DeepSeek Harness 构建了对智能体运行全生命周期进行确定性校验的自动化测试套件：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **AgentLoop State Machine** | 决策循环时序、工具调用有序派发、超步数熔断 | Pytest + Asyncio | 内存轻量模拟，隔离外部网络 |
| **Fault Injection WireMock** | 24 种网络故障（HTTP 502/504、TCP Reset、畸形 JSON、SSE 延迟切片） | WireMock / Respx | 本地端口代理劫持 |
| **Sandbox Execution Lifecycle**| 子进程派生、超时自动 SIGKILL、资源限制（内存/CPU） | Pytest + Subprocess (Linux/Win) | 进程组树追踪（Process Tree kill） |
| **Snapshot Migration Tests** | 会话快照从 v1 格式向 v3 格式无损迁移、增量 Diff 校验 | Pytest + Pydantic Schema | 历史 Golden 快照数据集回归 |
| **Binary Smoke Tests** | 全静态编译单二进制文件运行、环境变量注入、基础 CLI 命令 | Bash / PowerShell 自动化脚本 | 隔离的临时操作系统目录 |
| **Headless Browser Replay** | Agent 在无头浏览器环境中的操作路径与时钟回放 | Playwright (Chromium/WebKit) | 固定虚拟系统时钟（Fake Timers） |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：AgentLoop 决策时序与状态机熔断控制测试

#### 3.1.1 业务功能与测试目标
智能体执行任务时遵循严格的循环：`Input -> Think -> Tool Call -> Observation -> Output`。
测试必须保证：
1. 工具调用（Tool Call）必须按照依赖顺序串行或受控并行派发，不允许乱序执行。
2. 当任务执行达到 `max_steps` 或遭遇不可逆异常时，状态机必须安全跃迁到 `STOPPED`，并输出完整的轨迹日志（Trajectory）。

#### 3.1.2 测试用例实现
```python
import pytest
import asyncio
from harness.core.loop import AgentLoop, LoopStatus
from harness.core.types import ToolCall, Observation

@pytest.mark.asyncio
async def test_agent_loop_execution_order_and_step_limit():
    execution_trace = []

    async def mock_tool_executor(tool_call: ToolCall) -> Observation:
        execution_trace.append(f"call_{tool_call.name}_{tool_call.call_id}")
        await asyncio.sleep(0.01)
        return Observation(call_id=tool_call.call_id, result="ok")

    loop = AgentLoop(
        tool_executor=mock_tool_executor,
        max_steps=3,
        timeout_seconds=5
    )

    # 1. 模拟 Agent 连续产生多次决策调用
    calls_batch = [
        ToolCall(call_id="1", name="search", args={"query": "test1"}),
        ToolCall(call_id="2", name="read_file", args={"path": "a.txt"}),
    ]

    status = await loop.run_step(calls_batch)
    assert status == LoopStatus.CONTINUE
    assert execution_trace == ["call_search_1", "call_read_file_2"]

    # 2. 模拟持续步数递增直至触发硬熔断
    for _ in range(3):
        await loop.run_step([ToolCall(call_id="x", name="ping", args={})])

    # 3. 严格断言：达到步数限制后状态机必须翻转为 STOPPED
    assert loop.status == LoopStatus.MAX_STEPS_EXCEEDED
    assert loop.current_step == 3
```

---

### 3.2 模块二：WireMock 24 种传输层与协议层故障注入测试

#### 3.2.1 业务功能与测试目标
大模型通过 HTTP/SSE 与外界通信极易受到网络波动影响。测试必须覆盖以下典型故障场景：
1. **传输层故障**：连接拒绝、TCP 连接建立后静默重置（RST）、高丢包延迟。
2. **HTTP 协议层故障**：服务端 500/502/503/504 错误、429 限流重试头（`Retry-After: 2`）。
3. **内容层故障**：Truncated SSE 数据包（数据传输中途截断）、非合法 UTF-8 字符流。

#### 3.2.2 测试用例实现
```python
import pytest
import httpx
from harness.fault_injection import WireFaultInjector, FaultType

@pytest.mark.asyncio
async def test_network_fault_injection_sse_truncation_and_recovery():
    # 1. 启动本地故障注入代理
    injector = WireFaultInjector()
    await injector.start()

    # 注入规则：在传输第 3 个 SSE chunk 时强制斩断 TCP 连接
    injector.inject_rule(
        route="/v1/chat/completions",
        fault=FaultType.TRUNCATE_STREAM_AT_CHUNK,
        param={"chunk_index": 3}
    )

    client = httpx.AsyncClient(base_url=injector.proxy_url)

    # 2. 验证 Harness 客户端在断流发生时的异常捕获与退避重连机制
    received_chunks = []
    connection_dropped = False

    try:
        async with client.stream("POST", "/v1/chat/completions", json={"prompt": "hi"}) as resp:
            async for chunk in resp.aiter_lines():
                received_chunks.append(chunk)
    except httpx.RemoteProtocolError:
        connection_dropped = True

    # 3. 强断言：确实在第 3 个 chunk 发生物理断连
    assert connection_dropped is True
    assert len(received_chunks) <= 3

    await injector.stop()
```

---

### 3.3 模块三：沙箱 Subprocess 树追踪与资源超时强制 Teardown 测试

#### 3.3.1 业务功能与测试目标
智能体在运行 Shell 命令时可能创建深层嵌套的子进程（例如 `bash -> python -> subprocess`）。当超时发生时，单纯对主进程发送 `SIGTERM` 会导致孤儿进程驻留。测试必须验证：
1. 基于进程组（Process Group / Job Object）实现全树递归追踪。
2. 超时发生后，所有子孙进程在 500ms 内被彻底物理终止。

#### 3.3.2 测试用例实现
```python
import pytest
import time
import psutil
from harness.sandbox.process import ProcessSandbox

def test_sandbox_kills_entire_subprocess_tree_on_timeout():
    sandbox = ProcessSandbox(timeout=2)

    # 启动一个派生深层子进程的 Shell 任务
    command = "bash -c 'python -c \"import time; time.sleep(100)\" & python -c \"import time; time.sleep(100)\" & wait'"
    
    start_time = time.time()
    result = sandbox.execute(command)
    duration = time.time() - start_time

    # 1. 耗时断言：必须在设定超时阈值附近被强制截断
    assert 1.9 <= duration <= 3.0
    assert result.timed_out is True
    assert result.exit_code != 0

    # 2. 物理进程排查断言：测试机中不能有任何残留的 sleep 100 进程
    lingering_procs = [
        p for p in psutil.process_iter(["name", "cmdline"])
        if "time.sleep(100)" in " ".join(p.info.get("cmdline") or [])
    ]
    assert len(lingering_procs) == 0
```

---

### 3.4 模块四：会话快照（Session Snapshot）跨版本无损迁移测试

#### 3.4.1 业务功能与测试目标
评测任务可能历经数天，中途会持久化 Agent 记忆与执行上下文。测试必须验证：
1. 历史存储的 v1 格式快照（扁平字典）无缝升级到 v3 格式（强类型 Pydantic Schema，包含 token 消耗审计与分支追踪）。
2. 快照还原后，Agent 能够从中断的精确步骤继续向前执行，不产生重复操作。

#### 3.4.2 测试用例实现
```python
import pytest
from harness.snapshot.migration import migrate_snapshot_v1_to_v3
from harness.snapshot.schema import SnapshotV3

def test_session_snapshot_v1_to_v3_lossless_migration():
    v1_snapshot_data = {
        "session_id": "sess_legacy_999",
        "history": [
            {"role": "user", "text": "Deploy nginx"},
            {"role": "assistant", "text": "Running apt-get"}
        ],
        "created_at_timestamp": 1700000000
    }

    # 1. 执行模式升级迁移
    v3_snapshot = migrate_snapshot_v1_to_v3(v1_snapshot_data)

    # 2. 校验强类型实例属性
    assert isinstance(v3_snapshot, SnapshotV3)
    assert v3_snapshot.session_id == "sess_legacy_999"
    assert len(v3_snapshot.turns) == 2
    
    # 验证向后兼容字段被自动赋予安全默认值
    assert v3_snapshot.schema_version == "3.0.0"
    assert v3_snapshot.total_tokens_consumed == 0
    assert v3_snapshot.branch_id == "main"
```

---

## 4. 测试工程化与持续集成（CI）保障

1. **确定性无头时间同步（Deterministic Clock Driving）**：
   * 在涉及 Playwright 浏览器和网络请求重试的端到端用例中，注入基于 `libfaketime` 或虚拟时钟模块，消除时间波动造成的非确定性测试失败。
2. **轻量二进制编译冒烟流水线**：
   * 在跨平台（Linux x86_64 / macOS arm64 / Windows x64）编译产出单二进制文件后，自动挂载独立空容器执行 `--version` 与预置基准验证。

---

## 5. 对同类复杂系统的参考价值

* **多智能体沙箱的安全退出守则**：任何运行大模型自由指令的系统，必须将进程管理提升到操作系统进程组级，并在用例断言中加入物理进程扫描，杜绝测试环境被僵尸进程占满。
* **网络抖动的全覆盖容错矩阵**：不能仅测试 API 200 与 500，针对 SSE 流式切片中途中断的注入测试，是评估 AI 客户端健壮性的决定性环节。
