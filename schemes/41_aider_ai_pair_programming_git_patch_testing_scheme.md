# Aider 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[Aider](https://github.com/paul-gauthier/aider) 是全球公认工程化最极致的命令行 AI 结对编程工具（AI Pair Programming in Terminal）。Aider 支持在本地代码仓库中直接通过交互式对话完成代码重构、测试修复与功能编写。其核心技术亮点在于**基于 Tree-sitter 的代码库全景符号地图（Repository Map）、多模型自适应的代码编辑策略适配器（UDiff / Wholefile / Search-Replace Editblock）以及全自动的 Git 事务化提交与原子回退机制**。

```
┌────────────────────────────────────────────────────────────────────────┐
│             Interactive Terminal CLI (Prompt Toolkit + xterm)          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ 用户多模态/文本输入流
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Aider Core Engine (Python)                          │
├───────────────────────────────┬────────────────────────────────────────┤
│ Tree-sitter Repo Map Engine   │ EditBlock & Patch Applicator           │
│ - 全局 AST 语法树符号索引     │ - SEARCH/REPLACE 差异块智能对齐        │
│ - PageRank 算法重要度打分     │ - 自动缩进校准与空格容错               │
├───────────────────────────────┼────────────────────────────────────────┤
│ Git Transaction Manager       │ Voice & Multimodal Input Pipeline      │
│ - 自动生成规范 Commit Message │ - 本地 Whisper 语音实时转写            │
│ - 破坏性更改原子撤回 (/undo)  │ - 图片上下文提取与多模态解析           │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Aider 构建了保障代码安全写入与高精度代码定位的自动化测试套件：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Repo Map AST Unit Tests** | Tree-sitter 符号提取、类与函数签名提取、Token 预算裁剪 | Pytest + tree-sitter | 虚拟文件树，无需真实 Git 仓库 |
| **EditBlock Patch Tests** | SEARCH/REPLACE 模糊对齐、缩进修正、多块同时替换 | Pytest (`tests/basic/test_editblock.py`) | 纯字符串内存对比 |
| **Git Atomic Operations** | 自动 commit、`/undo` 还原工作区、脏工作树阻断 | Git CLI + Pytest (`tempfile`) | 独立临时 Git 仓库，用后销毁 |
| **Model Coder Adapter Tests** | 针对 GPT-4o、Claude 3.5 Sonnet、DeepSeek 的编辑策略选择 | Pytest + Mock LLM | 拦截 API 交互，比对 Prompt 模板 |
| **Interactive CLI Smoke Tests**| 模拟终端用户输入指令、补全响应、退出流程 | `pytest-mock` + Prompt Toolkit | 模拟 stdin/stdout 流 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：Tree-sitter Repo Map 符号提取与 PageRank 排名测试

#### 3.1.1 业务功能与测试目标
Aider 构建代码地图时，使用 Tree-sitter 提取类名、方法名与函数签名，并利用 PageRank 计算哪些文件是最核心的基础设施。测试必须保证：
1. 能够忽略文件内部冗长的具体实现细节，仅提取出准确的函数签名。
2. 严格受限于设定的 Token 预算（Token Budget），在达到上限时优先保留高排名符号。

#### 3.1.2 测试用例实现
```python
import pytest
from aider.repomap import RepoMap
from aider.io import InputOutput

def test_repo_map_extracts_clean_signatures_within_token_limit():
    io = InputOutput()
    repo_map = RepoMap(root="/test_workspace", io=io)

    python_source_code = """
class DataPipeline:
    \"\"\"Docstring to be preserved\"\"\"
    def __init__(self, db_url: str):
        self.db_url = db_url
        self._internal_state = 12345

    def execute_batch(self, items: list[dict]) -> bool:
        # A very long implementation that should be excluded
        res = []
        for x in items:
            res.append(x)
        return True
"""
    # 1. 提取符号地图
    extracted_map = repo_map.get_ranked_tags_map_from_content(
        fname="pipeline.py",
        content=python_source_code,
        max_map_tokens=100
    )

    # 2. 结构化严格断言：签名保留，函数体被裁掉
    assert "class DataPipeline:" in extracted_map
    assert "def execute_batch(self, items: list[dict]) -> bool:" in extracted_map
    assert "self._internal_state = 12345" not in extracted_map
    assert "for x in items:" not in extracted_map
```

---

### 3.2 模块二：SEARCH/REPLACE 差异块智能对齐与缩进自愈测试

#### 3.2.1 业务功能与测试目标
大模型输出的代码编辑块通常格式如下：
```
<<<<<<< SEARCH
    return False
=======
    return True
>>>>>>> REPLACE
```
在实际代码中，源文件的缩进可能是制表符（Tab）或不同的空格数量。Aider 必须能自动校正微小缩进偏差并精确替换。测试必须覆盖：
1. 包含空格缩进微小差异的自动匹配与自愈。
2. 当 SEARCH 块在文件中完全不匹配时抛出明确异常并拒绝写入。

#### 3.2.2 测试用例实现
```python
import pytest
from aider.coders.editblock_coder import find_original_update_blocks

def test_editblock_fuzzy_indentation_match_and_replacement():
    original_code = "def check_status():\n        # 8 spaces indent\n        return False\n"
    
    # 模拟 LLM 错误地只用了 4 个空格缩进输出 SEARCH 块
    llm_edit_block = """
<<<<<<< SEARCH
    # 8 spaces indent
    return False
=======
    # 8 spaces indent
    return True
>>>>>>> REPLACE
"""
    # 1. 执行智能匹配与替换
    success, new_code = find_original_update_blocks(
        edit_block=llm_edit_block,
        original_content=original_code,
        fuzzy=True
    )

    # 2. 强断言：替换成功且继承源文件原有的 8 个空格缩进
    assert success is True
    assert "        return True" in new_code
    assert "        return False" not in new_code
```

---

### 3.3 模块三：Git 自动提交与原子撤回（`/undo`）机制测试

#### 3.3.1 业务功能与测试目标
Aider 在每次 AI 修改文件后会自动生成一条 Git Commit。若用户不满意输入 `/undo`，Aider 必须自动回退至修改前并清理工作区。测试必须覆盖：
1. 自动提交包含规范前缀（如 `aider: update pipeline.py`）。
2. 执行 undo 命令后，工作区与暂存区物理恢复一致，无残留脏文件。

#### 3.3.2 测试用例实现
```python
import tempfile
import subprocess
import os
import pytest
from aider.repo import GitRepo
from aider.io import InputOutput

@pytest.fixture
def temp_git_workspace():
    with tempfile.TemporaryDirectory() as tmpdir:
        subprocess.run(["git", "init"], cwd=tmpdir, check=True)
        subprocess.run(["git", "config", "user.name", "Tester"], cwd=tmpdir, check=True)
        subprocess.run(["git", "config", "user.email", "test@aider.com"], cwd=tmpdir, check=True)
        
        file_path = os.path.join(tmpdir, "calc.py")
        with open(file_path, "w") as f:
            f.write("result = 0\n")
        
        subprocess.run(["git", "add", "."], cwd=tmpdir, check=True)
        subprocess.run(["git", "commit", "-m", "initial commit"], cwd=tmpdir, check=True)
        yield tmpdir

def test_git_auto_commit_and_undo_rollback(temp_git_workspace):
    repo = GitRepo(InputOutput(), [temp_git_workspace], None)

    # 1. 模拟 AI 写入新内容
    file_path = os.path.join(temp_git_workspace, "calc.py")
    with open(file_path, "w") as f:
        f.write("result = 42\n")

    # 2. 触发自动提交
    repo.auto_commit("feat: update result to 42", [file_path])
    
    # 验证 Commit 记录存在
    last_commit_msg = subprocess.check_output(
        ["git", "log", "-1", "--pretty=%B"], cwd=temp_git_workspace
    ).decode("utf-8")
    assert "feat: update result to 42" in last_commit_msg

    # 3. 触发 Undo 回退
    repo.undo_last_commit()

    # 4. 物理断言：文件内容彻底回滚到初始状态
    with open(file_path) as f:
        assert f.read() == "result = 0\n"
```

---

## 4. 测试工程化与基础设施

1. **零外部网络 Mock 策略**：
   * 在针对不同模型策略（Wholefile vs Editblock）进行测试时，全线使用本地预设的 LLM 回复桩，杜绝因外部网络故障或 API 欠费导致测试失败。
2. **纯内存虚拟 Git 仓库隔离**：
   * 采用 `tempfile.TemporaryDirectory` 在 RAM 磁盘上秒级创建销毁 Git 仓库，每秒可运行上百个带版本控制的集成测试。

---

## 5. 对同类复杂系统的参考价值

* **非结构化输出的结构化自愈算法**：大语言模型生成格式代码经常存在微小偏差（缩进差异、换行符 `\r\n` vs `\n`）。编写全面的模糊对齐与自愈单测，是保证 AI 代码写入物理磁盘可靠性的关键基石。
