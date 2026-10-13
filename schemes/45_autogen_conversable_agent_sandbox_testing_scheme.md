# AutoGen 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[AutoGen](https://github.com/microsoft/autogen) 是微软开源的全球顶尖多智能体对话编排与自主代码执行框架。系统基于 Python 构建，创新提出了以 **ConversableAgent（可对话智能体）** 为核心的事件驱动状态机，支持通过对话形式实现智能体与智能体（Agent-to-Agent）、智能体与人类（Human-in-the-Loop）以及智能体与代码执行环境（UserProxyAgent）的高阶协同，并支持群聊（GroupChat）动态发言人调度。

### 1.2 被测系统核心组成
1. **ConversableAgent 对话状态机（Chat State Machine）**：管理多轮消息队列、自动回复触发器（Auto-Reply Triggers）与钩子函数（Hooks）。
2. **代码沙箱执行器（Code Execution Command Runner）**：自动提取 Markdown 中的代码块，在本地子进程或 Docker 隔离容器中物理执行并回传终端结果。
3. **GroupChat 动态发言人调度器（Speaker Selection Engine）**：基于轮询（Round-Robin）、自定义函数或群聊经理（GroupChatManager LLM）决定下一发言人。
4. **会话终止与人机交互控制器（Termination & Human Input Mode）**：支持 `ALWAYS`, `TERMINATE`, `NEVER` 模式，识别终止关键词并平滑退出。

### 1.3 自动化测试的主要难点与挑战
* **智能体对话循环的无限递归**：当终止条件不明确时，两个 Agent 可能持续互相寒暄导致测试超时。
* **物理代码执行的安全与文件残留**：自动运行 Python/Bash 代码时的沙箱逃逸防护与临时目录清理。
* **群聊中发言人顺序的非确定性校验**：动态选择发言人模式下的状态收敛性测试。

---

## 2. 测试架构全景与分层策略

AutoGen 建立了以受控对话拦截与容器化代码沙箱为核心的自动化测试网：

```
┌────────────────────────────────────────────────────────┐
│             GroupChat Multi-Agent E2E Scenarios        │  群聊全景层：多智能体协作解决复杂数学/编程问题全链路
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Docker & Local Subprocess Code Execution Tests    │  代码沙箱层：物理提取并执行 Python 脚本，验证退出码
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      ConversableAgent Auto-Reply & Termination Suite   │  状态机核心层：测试消息流转、终止词判定与人机模式切换
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Cache & Mock LLM Replay Testbed                │  桩与缓存层：基于磁盘 DiskCache 确定性回放模型交互
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：UserProxyAgent 代码提取、沙箱执行与结果回传闭环

#### 3.1.1 业务功能背景
AssistantAgent 输出了 ````python\nprint(100 + 200)\n````。UserProxyAgent 收到该消息后，自动识别并提取该 Python 脚本，在指定工作区目录启动沙箱执行，并将控制台标准输出 `300` 作为下一条消息发回给 AssistantAgent。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 真实沙箱执行测试）**：
```python
import pytest
from autogen import AssistantAgent, UserProxyAgent

def test_user_proxy_executes_extracted_code_and_returns_output(tmp_path):
    # 1. 构造带有物理沙箱工作区的 UserProxyAgent
    user_proxy = UserProxyAgent(
        name="user_proxy",
        human_input_mode="NEVER",
        code_execution_config={"work_dir": str(tmp_path), "use_docker": False},
        max_consecutive_auto_reply=1
    )

    # 2. 模拟 Assistant 发送带有可执行代码块的消息
    code_message = """
Here is the solution:
```python
with open("result.txt", "w") as f:
    f.write("COMPUTED_SUCCESSFULLY")
print("FINISHED")
```
"""

    # 3. 触发 UserProxy 的自动回复（执行代码）
    reply = user_proxy.generate_reply(messages=[{"role": "assistant", "content": code_message}])

    # 4. 严格双重断言：
    # 逻辑回执断言：终端输出必须包含 FINISHED
    assert "FINISHED" in reply
    assert "exitcode: 0" in reply
    # 真实物理世界副作用断言：磁盘上的文件必须真实生成且内容匹配
    assert (tmp_path / "result.txt").read_text() == "COMPUTED_SUCCESSFULLY"
```

---

### 3.2 功能二：GroupChat 会话终止条件（Termination Trigger）判定

#### 3.2.1 业务功能背景
群聊配置了终止判定函数：当任意 Agent 发言内容以 `"TERMINATE"` 结尾时，整个群聊会话必须立即终止，群经理不再唤醒任何新发言人。

#### 3.2.2 测试用例设计方案
* **测试编排**：
  1. 编排 3 个 Agent 进入 GroupChat。
  2. 模拟第 2 轮对话中某个 Agent 输出 `"All tasks are finished. TERMINATE"`。
  3. **断言**：群聊控制器立即中断，总发言次数精确等于 2，未触发第 3 轮发言，返回状态为成功终止。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **内置 DiskCache 缓存确定性回放**：
   * AutoGen 提供基于 SQLite 的调用缓存，首次录制后在 CI 中强制开启缓存命中模式，实现零网络、零延迟回归。
2. **进程泄漏看门狗**：
   * 在执行本地子进程代码测试后，测试夹具通过 `psutil` 检查是否存在挂起 Python 子进程，发现孤儿进程立即强制终止。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **将“代码提取与物理执行”作为独立测试链条**：对于具备代码执行能力的 Agent，把“Markdown 中提取多语言脚本的健壮性”与“沙箱真实执行”解耦测试，能极大提升排错效率。
* **对话驱动型系统必须有“确定性终止判定”**：在设计多智能体对话时，绝不能依赖模型自发停止，必须在测试用例中建立严格的硬性终止条件与最大轮次熔断。
