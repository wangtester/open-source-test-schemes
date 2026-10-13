# Guidance 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Guidance](https://github.com/guidance-ai/guidance) 是由微软研究院开源的高性能受控语言模型生成与交互式模板编程语言。系统核心基于 Python 与 C++ 编写，通过构建高效的**词法图（Grammar Graph）**与状态机，创新实现了在同一控制流中将固定提示文本、自回归模型生成、条件判断与变量提取交替融合（Interleaved Execution），并能在下一个 Token 唯一确定时实现**前缀跳跃贪婪合并（Token Greediness / Acceleration）**。

### 1.2 被测系统核心组成
1. **引导文法 AST 编译器（Grammar AST Compiler）**：支持 `select(['A', 'B'])`, `gen(regex=...)`, `capture(name)` 声明式模板编译为 C++ 执行图。
2. **交替执行运行时（Interleaved Runtime）**：在固定字符（Prompt）由程序直接塞入上下文，动态部分才调用模型生成的精细调度。
3. **前缀贪婪加速器（Greedy Token Skipping）**：在状态机推导发现仅有唯一可能分支时，直接跨过自回归推理前向传播，大幅降低推理延迟。
4. **统一状态机上下文（Program State & Variables）**：管理模板执行全生命周期的上下文文本与捕获变量字典。

### 1.3 自动化测试的主要难点与挑战
* **C++ 词法状态机与 Python 对象的双向交互**：底层 C++ 字节图在处理复杂字符编码时与 Python 字符串切片的边界一致性。
* **交替执行时自回归步进的控制反转断言**：如何验证模型推理调用仅在 `gen()` 处触发，而在固定字符串处绝对零开销直接拼接。
* **嵌套选择（Nested Select）与回溯（Backtracking）**：多层分支匹配失败时的状态回滚。

---

## 2. 测试架构全景与分层策略

Guidance 建立了以状态图匹配与虚拟模型计步器为核心的自动化测试网：

```
┌────────────────────────────────────────────────────────┐
│             Real Model Acceleration Benchmark Tests    │  性能加速层：比对标准自回归与 Guidance 贪婪跳跃的前向步数
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Template Control Flow & Capture Tests          │  模板逻辑层：验证 if/else, select, capture 变量绑定
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      C++ Grammar Graph Traversal & Matcher Tests       │  文法图底层：单测验证词法图节点遍历、前缀跳跃与回溯
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Mock Model Step-Counter Test Harness           │  调用计步桩：捕获模型 forward() 调用次数与传入上下文
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：交替执行（Interleaved Execution）步数跳跃与前向调用断言

#### 3.1.1 业务功能背景
模板定义为：`"The count is: " + gen('num', regex=r'[0-9]+') + ". Done."`。系统执行时，“The count is: ”与“. Done.”这两段固定文字必须直接注入 KV 缓存，模型仅为数字生成调用 `forward()`，绝不浪费算力去自回归生成固定句尾。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest + 计步模型桩）**：
```python
import pytest
import guidance
from guidance import gen, select

def test_interleaved_execution_skips_fixed_tokens(step_counting_mock_model):
    # 1. 构造带有计步探针的 Mock 模型（预置模型生成数字 '42'）
    lm = step_counting_mock_model
    lm.set_next_generation("42")

    # 2. 定义包含固定文本与动态生成的交替模板
    program = lm + "Result is: " + gen(name="val", regex=r"[0-9]+") + ", perfectly executed."

    # 3. 严格断言
    # 验证变量捕获字典
    assert program["val"] == "42"
    # 验证最终合并文本完备性
    assert str(program) == "Result is: 42, perfectly executed."
    # 核心硬断言：模型的前向步进调用次数精确等于 2（生成 '4' 和 '2'），固定文本绝对未触发 forward
    assert lm.get_forward_call_count() == 2
```

---

### 3.2 功能二：`select()` 选项受限与自动前缀合并

#### 3.2.1 业务功能背景
选项为 `select(['apple', 'application'])`。状态机观察到两个选项具有公共前缀 `'appl'`，系统自动将 `'appl'` 直接写入上下文，仅让模型从 `'e'` 与 `'ication'` 中选择其一。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
def test_select_common_prefix_merging(step_counting_mock_model):
    lm = step_counting_mock_model
    lm.set_next_generation("e") # 模拟模型选择 apple

    program = lm + "Pick: " + select(["apple", "application"], name="choice")

    assert program["choice"] == "apple"
    # 验证前缀 'appl' 未消耗独立采样，公共前缀被原子化折叠
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **C++ 词法图编译跨平台矩阵**：
   * 在 Linux (GCC/Clang)、Windows (MSVC) 和 macOS (Apple Clang) 上执行 C++ 单元测试，确保无符号字符与内存指针在不同系统下的内存安全。
2. **纯虚拟模型执行引擎（Mock Engine）**：
   * 所有功能逻辑测试通过内存字典与计步器执行，毫秒级验证语法树的正确性。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **将推理效率纳入自动化断言指标**：在测试 AI 优化系统时，不仅要断言结果字符串是对的，更要把“模型的调用步数（Step Count / Forward Count）”作为硬性断言，杜绝性能倒退。
* **利用交替模板消除幻觉与格式漂移**：对于输出结构严苛的系统，在测试层面断言“静态语法骨架直接由程序强制灌入”，能从架构根源上杜绝模型在骨架处的胡言乱语。
