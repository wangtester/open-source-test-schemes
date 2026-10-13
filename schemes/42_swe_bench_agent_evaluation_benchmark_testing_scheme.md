# SWE-bench 自动化测试方案与评估基准用例全集深度剖析

## 1. 项目背景与系统架构全景

[SWE-bench](https://github.com/princeton-nlp/SWE-bench) 是全球学术界与工业界通用的衡量 AI 软件工程能力的最权威基准平台。它收集了数千个来自真实知名开源 Python 仓库（如 Django、SymPy、Flask、Requests 等）中解决过的真实 GitHub Issue 与 Pull Request。

SWE-bench 不仅是一个静态数据集，更是一个高并发、多沙箱的**端到端自动化测试评估执行引擎**。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SWE-bench Evaluation CLI                        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ 多进程任务分派 (multiprocessing / Ray)
┌───────────────────────────────────▼────────────────────────────────────┐
│                    SWE-bench Execution Engine                          │
├───────────────────────────────┬────────────────────────────────────────┤
│ Environment & Image Builder   │ Patch Applicator & Git Controller      │
│ - 根据实例 ID 动态匹配基础镜像 │ - 提取 Agent 生成的 Git Diff 补丁       │
│ - Conda/Pip 依赖环境缓存隔离  │ - 执行 git apply 物理打补丁与状态还原  │
├───────────────────────────────┼────────────────────────────────────────┤
│ Pytest/Unit Test Runner       │ Dual-Matrix Resolution Judge           │
│ - 注入官方隐藏测试用例 (Eval) │ - FAIL_TO_PASS (原先失败，打补丁后通过)│
│ - 收集 JUnit XML/日志输出     │ - PASS_TO_PASS (原先通过，不能被破坏)  │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

SWE-bench 体系本身的测试套件与执行机制涵盖以下层次：

| 测试层级 | 被测目标与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Dataset Schema Tests** | 验证 Dataset JSONL 结构、Commit Hash 存在性、测试列表完整性 | Pytest (`tests/test_dataset.py`) | 纯静态校验，秒级完成 |
| **Git Patch Engine Tests** | Git Diff 解析、Whitespace 容错、多层级补丁应用、冲突回退 | Git CLI + Pytest | 临时创建临时 Git 仓库，运行后物理删除 |
| **Test Log Parsing Tests** | 解析 Pytest、Unittest、Django Runner 等不同框架的终端输出 | 正则与 AST 文本测试 | 各测试框架的原始日志快照断言 |
| **Docker Sandbox Isolation** | 动态拉起指定实例容器、限制 CPU/内存、禁止外部网络防作弊 | Docker API + `pytest-asyncio` | 容器挂载只读根目录，销毁回收 |
| **Evaluation Matrix Judge** | 验证双向矩阵算法判定准确性（Resolving Criteria） | 单元测试构造测试用例状态树 | 模拟矩阵输入，验证判定真值表 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：评测判题双向矩阵（FAIL_TO_PASS & PASS_TO_PASS）逻辑测试

#### 3.1.1 业务功能与测试目标
SWE-bench 衡量一个补丁是否真正“解决问题且无副作用”的核心标准是**双向状态矩阵判定**：
1. **FAIL_TO_PASS**：必须将原本因 Bug 报错的用例集全部转为 `PASSED`。
2. **PASS_TO_PASS**：原本无 Bug 的历史功能用例必须继续全部保持 `PASSED`，不能出现任何功能退化（Regression）。
3. 只要出现任何一项未满足，该 Issue 的解决状态判定为 `RESOLVED = False`。

#### 3.1.2 测试用例实现
```python
import pytest
from swebench.harness.grading import compute_fail_to_pass, compute_pass_to_pass, get_eval_report

def test_evaluation_judge_strict_resolution_criteria():
    instance_meta = {
        "instance_id": "django__django-11099",
        "FAIL_TO_PASS": ["tests.test_auth.test_token_expiry", "tests.test_auth.test_invalid_signature"],
        "PASS_TO_PASS": ["tests.test_auth.test_login_success", "tests.test_auth.test_logout"]
    }

    # 场景 1：完全修复且无退化 -> PASS
    run_log_perfect = {
        "tests.test_auth.test_token_expiry": "PASSED",
        "tests.test_auth.test_invalid_signature": "PASSED",
        "tests.test_auth.test_login_success": "PASSED",
        "tests.test_auth.test_logout": "PASSED"
    }
    report_perfect = get_eval_report(instance_meta, run_log_perfect)
    assert report_perfect["resolved"] is True
    assert report_perfect["fail_to_pass_success"] is True
    assert report_perfect["pass_to_pass_success"] is True

    # 场景 2：修复了 Bug，但破坏了原有的 login_success -> FAIL (引入了功能退化)
    run_log_regression = {
        "tests.test_auth.test_token_expiry": "PASSED",
        "tests.test_auth.test_invalid_signature": "PASSED",
        "tests.test_auth.test_login_success": "FAILED", # 致命退化
        "tests.test_auth.test_logout": "PASSED"
    }
    report_regression = get_eval_report(instance_meta, run_log_regression)
    assert report_regression["resolved"] is False
    assert report_regression["pass_to_pass_success"] is False

    # 场景 3：未修全，其中一个目标测试依旧失败 -> FAIL
    run_log_incomplete = {
        "tests.test_auth.test_token_expiry": "PASSED",
        "tests.test_auth.test_invalid_signature": "FAILED",
        "tests.test_auth.test_login_success": "PASSED",
        "tests.test_auth.test_logout": "PASSED"
    }
    report_incomplete = get_eval_report(instance_meta, run_log_incomplete)
    assert report_incomplete["resolved"] is False
    assert report_incomplete["fail_to_pass_success"] is False
```

---

### 3.2 模块二：Git 补丁提取、物理应用与原子回滚测试

#### 3.2.1 业务功能与测试目标
评测时需要从 Agent 响应中提取 Git Diff 并在仓库检出分支上运行 `git apply`。测试必须覆盖：
1. 正常 Patch 能够正确修改代码行。
2. 脏补丁（如包含 Markdown 标记、未闭合的 hunk、路径前缀缺失 `a/` `b/`）的容错修复能力。
3. 补丁应用失败时能够自动 `git reset --hard` 回归干净状态，防止污染后续运行。

#### 3.2.2 测试用例实现
```python
import subprocess
import tempfile
import os
import pytest
from swebench.harness.utils import apply_patch

@pytest.fixture
def temp_git_repo():
    with tempfile.TemporaryDirectory() as tmpdir:
        # 初始化一个轻量 Git 临时仓库
        subprocess.run(["git", "init"], cwd=tmpdir, check=True)
        subprocess.run(["git", "config", "user.name", "Tester"], cwd=tmpdir, check=True)
        subprocess.run(["git", "config", "user.email", "test@test.com"], cwd=tmpdir, check=True)
        
        file_path = os.path.join(tmpdir, "main.py")
        with open(file_path, "w") as f:
            f.write("def calculate():\n    return 10\n")
        
        subprocess.run(["git", "add", "."], cwd=tmpdir, check=True)
        subprocess.run(["git", "commit", "-m", "init"], cwd=tmpdir, check=True)
        yield tmpdir

def test_apply_patch_success_and_atomic_rollback(temp_git_repo):
    # 1. 应用正常补丁
    valid_patch = """diff --git a/main.py b/main.py
--- a/main.py
+++ b/main.py
@@ -1,2 +1,2 @@
 def calculate():
-    return 10
+    return 42
"""
    success = apply_patch(temp_git_repo, valid_patch)
    assert success is True

    # 验证物理文件内容修改
    with open(os.path.join(temp_git_repo, "main.py")) as f:
        assert f.read() == "def calculate():\n    return 42\n"

    # 2. 应用错误冲突补丁并触发回滚
    conflicted_patch = """diff --git a/main.py b/main.py
--- a/main.py
+++ b/main.py
@@ -1,2 +1,2 @@
-def non_existent_function():
+def replace():
"""
    fail_res = apply_patch(temp_git_repo, conflicted_patch)
    assert fail_res is False

    # 验证回滚：仓库状态依然保持在有效补丁状态，没有留下 .rej 脏文件
    assert not os.path.exists(os.path.join(temp_git_repo, "main.py.rej"))
```

---

### 3.3 模块三：多框架测试输出日志解析器（Log Parser）鲁棒性测试

#### 3.3.1 业务功能与测试目标
不同 Python 框架的测试控制台输出格式完全不同（Pytest 使用点号与 F，Unittest 打印 Traceback，Django 使用定制 runner）。测试必须验证：
1. 提取每个具体用例的终态（`PASSED`、`FAILED`、`ERROR`、`SKIPPED`）。
2. 在发生 Segmentation Fault、内存超限或系统崩溃时的非正常日志提取。

#### 3.3.2 测试用例实现
```python
from swebench.harness.log_parsers import parse_log_pytest, parse_log_unittest

def test_parse_pytest_log_with_ansi_and_summary():
    pytest_output = """
============================= test session starts ==============================
rootdir: /workspace
collected 3 items

tests/test_demo.py::test_ok PASSED                                      [ 33%]
tests/test_demo.py::test_fail FAILED                                    [ 66%]
tests/test_demo.py::test_skip SKIPPED (reason)                          [100%]

=================================== FAILURES ===================================
__________________________________ test_fail ___________________________________
    def test_fail():
>       assert 1 == 2
E       assert 1 == 2
=========================== short test summary info ============================
FAILED tests/test_demo.py::test_fail - assert 1 == 2
==================== 1 failed, 1 passed, 1 skipped in 0.12s ====================
"""
    parsed_results = parse_log_pytest(pytest_output)

    assert parsed_results["tests/test_demo.py::test_ok"] == "PASSED"
    assert parsed_results["tests/test_demo.py::test_fail"] == "FAILED"
    assert parsed_results["tests/test_demo.py::test_skip"] == "SKIPPED"
```

---

### 3.4 模块四：Docker 容器隔离执行与防作弊网络封锁测试

#### 3.4.1 业务功能与测试目标
为了防止 Agent 偷偷联网下载现成的补丁，或者利用本地进程提权破坏评测机，SWE-bench 必须保证：
1. 容器运行评测阶段将网络模式设置为 `none`（完全断网）。
2. 设置强制超时上限，防止死锁或死循环用例阻塞评测集群。

#### 3.4.2 测试用例实现
```python
import docker
import pytest
from swebench.harness.docker_build import run_instance_eval

@pytest.mark.asyncio
async def test_docker_eval_enforces_network_isolation_and_timeout():
    client = docker.from_env()

    # 尝试在评测容器内部发起 curl 网络请求
    test_cmd = "python -c 'import urllib.request; urllib.request.urlopen(\"https://www.google.com\", timeout=2)'"
    
    container = client.containers.run(
        image="swebench/eval-base:latest",
        command=f"bash -c \"{test_cmd}\"",
        network_mode="none", # 关键：完全物理断网
        detach=True
    )

    result = container.wait(timeout=10)
    container_logs = container.logs().decode("utf-8")
    container.remove(force=True)

    # 必须报错：无法连接外部网络
    assert result["StatusCode"] != 0
    assert "URLError" in container_logs or "Network is unreachable" in container_logs
```

---

## 4. 测试工程化与基础设施

1. **镜像分层预热缓存（Cashing System）**：
   * SWE-bench 针对每个 Python 库预构建基础镜像（如 `sweb.env.django_11099`），测试前只挂载特定补丁，将单实例评测启动时间从 10 分钟压缩至 15 秒。
2. **多进程与 Ray 任务调度**：
   * 采用分布式调度避免单机 CPU 打满，并在每个 Task 退出时执行垃圾回收（Docker Prune）。

---

## 5. 对同类复杂系统的参考价值

* **双矩阵评估法则（Dual-Matrix Judging）**：任何自动修复、代码生成或规则重构系统，必须同时制定“修复目标通过率”与“全量存量回归不降级率”，二者兼备才是有效交付。
* **隔离评测环境的安全性**：运行不受信任的 AI 自动化代码时，严格的只读挂载与网络阻断是防止环境投毒的最高准则。
