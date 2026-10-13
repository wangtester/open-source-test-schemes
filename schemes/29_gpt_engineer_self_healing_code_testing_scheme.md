# gpt-engineer 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[gpt-engineer](https://github.com/gpt-engineer-org/gpt-engineer) 是开源自主软件构建与代码自愈（Self-Healing）领域的先驱项目。系统代码演进深度依托 AI 智能体“自举生成（Dogfooding）”。系统核心由 Python 编写，前端配备现代交互式 Web 构建控制台，核心能力是根据用户自然语言需求，自主规划文件拓扑、生成完整代码库、执行验证并根据运行报错执行闭环自愈修复。

### 1.2 被测系统核心组成
1. **代码生成与文件系统树解析（Code Generation & File Tree Parser）**：将 LLM 输出的非结构化 Markdown/代码块自动解析为物理多层级文件树。
2. **执行验证与自愈循环（Execution & Self-Healing Loop）**：在受限子进程中自动安装依赖并执行入口代码，捕获报错堆栈并指导下一轮修复。
3. **前置需求澄清向导（Clarification Wizard & Specs）**：在生成前通过多轮结构化问答消除需求歧义。
4. **Web 交互式实时大盘**：呈现文件生成进度、Diff 代码对比、实时终端日志输出。

### 1.3 自动化测试的主要难点与挑战
* **生成代码的多样性与物理执行风险**：生成的代码可能包含不可控的无限循环或危险系统调用。
* **自愈闭环的多轮非确定性**：自愈过程需要经历“生成 -> 报错 -> 修补 -> 再运行”多轮状态机推进。
* **文件解析器对脏 Markdown 标记的健壮性**：LLM 输出的代码块可能缺失闭合标记或路径注解错误。

---

## 2. 测试架构全景与分层策略

gpt-engineer 演进出了以物理执行验证与解析器容错为核心的测试架构：

```
┌────────────────────────────────────────────────────────┐
│             Web UI & Diff Viewer Playwright Tests      │  前端交互层：驱动需求输入向导、代码 Diff 预览与重试
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Self-Healing Loop Subprocess Execution Tests      │  自愈验证层：故意注入含 Bug 代码，验证自动修复与退出码
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         File Parser & Markdown Extractor Unit Tests    │  解析纯单元层：测试从畸形大模型文本中提取文件路径与代码
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Temporary Directory File Fixture System        │  数据底座：每次生成在独立 tmpdir 隔离运行，自动销毁
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：代码执行报错自愈闭环（Self-Healing Loop）测试

#### 3.1.1 业务功能背景
系统初次生成的 Python 代码中包含一个语法错误（如引用了未定义的变量 `undefined_var`）。自愈引擎执行该代码失败后，自动收集 Traceback，送回模型生成补丁，并执行第二次运行。系统必须在重试预算内达成测试通过（退出码 0）。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 闭环集成测试）**：
```python
import pytest
from gpt_engineer.core.default.disk_execution_env import DiskExecutionEnv
from gpt_engineer.core.default.steps import self_heal

def test_self_healing_fixes_syntax_error(tmp_path):
    # 1. 构造带有缺陷的代码文件
    env = DiskExecutionEnv(tmp_path)
    env.write("main.py", "def run():\n    print(undeclared_variable)\nrun()")

    # 2. 模拟 LLM 在收到报错后提供修复版代码
    mock_ai = MockAI(responses=[
        "Fixing the bug:\n```python\nmain.py\ndef run():\n    print('FIXED_SUCCESSFULLY')\nrun()\n```"
    ])

    # 3. 运行自愈逻辑
    success, log = self_heal(mock_ai, env, entrypoint="python main.py", max_retries=2)

    # 4. 严格物理验证：自愈成功，退出码为 0，终端日志包含预期文本
    assert success is True
    assert "FIXED_SUCCESSFULLY" in log
    # 验证物理文件内容已被重写修复
    assert "undeclared_variable" not in env.read("main.py")
```

---

### 3.2 功能二：非规范 Markdown 代码块多文件解析提取测试

#### 3.2.1 业务功能背景
大模型可能以不同风格输出文件（如 `### src/app.js` 或在代码块首行添加注释 `// filename: src/app.js`）。解析器必须具备强大的鲁棒性，精准将各文件剥离并还原相对路径目录树。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
from gpt_engineer.core.default.disk_memory import parse_chat_into_file_list

def test_parse_chat_extracts_nested_files():
    chat_text = """
Here is the project:
`src/index.js`
```javascript
console.log('index');
```

`config/settings.json`
```json
{"port": 8080}
```
"""
    files = parse_chat_into_file_list(chat_text)

    # 断言提取出的文件名与代码内容
    assert len(files) == 2
    assert files[0].path == "src/index.js"
    assert "console.log('index');" in files[0].content
    assert files[1].path == "config/settings.json"
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **子进程执行超时看门狗（Process Timeout Guard）**：
   * 所有针对生成代码的执行测试，必须由 `subprocess.run(timeout=10)` 强行看护，防止生成的代码包含无限 `while True` 导致 CI 挂起。
2. **轻量沙箱文件比对断言**：
   * 使用 `pytest` 的 `tmp_path` fixture，每个用例运行在物理隔离的操作系统临时路径中，杜绝测试间文件冲突。

---

## 5. 对复杂系统的实战借鉴意义

* **自愈/重试机制必须设计“步数受限与收敛断言”**：带有自动修复、异常重试逻辑的软件，测试重点是验证其能否在限定步数内收敛，而不是无限制消耗资源。
* **代码生成类工具的底座是“物理执行验证（Verify by Running）”**：不看代码写得好看与否，以真实子进程跑起来的退出码为唯一的黄金断言标准。
