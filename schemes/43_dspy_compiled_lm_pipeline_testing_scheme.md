# DSPy 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[DSPy](https://github.com/stanfordnlp/dspy) 是由斯坦福大学 NLP 团队提出的颠覆性大模型编程与自动化优化编译框架。它将传统脆弱、手写的自然语言 Prompt 工程，升华为了声明式的**模块化编程与参数优化编译（Programming, not prompting）**。系统基于 Python 构建，提供强大的 Signature 声明规范、模块抽象（Predict, ChainOfThought）与提示词自动编译器（Teleprompters / Optimizers）。

### 1.2 被测系统核心组成
1. **声明式签名引擎（Signatures & Fields）**：通过类型与描述声明输入（InputField）与输出（OutputField），自动生成结构化 Prompt 契约。
2. **提示词自动编译器（Optimizers / Teleprompters）**：包括 `BootstrapFewShot`, `MIPRO`, `COPRO`，通过指标打分（Metric）在训练样本上自动化合成高质量思维链（CoT）示范样本与最优指令。
3. **推理模块管道（Predictors & ChainOfThought）**：标准化的多步推理链路组装器，支持类似 PyTorch `nn.Module` 的分层拓扑。
4. **编译前后指标评估器（Evaluator & Metrics）**：在验证集上自动化计算端到端评估指标（如准确率、F1 分数、语义相关度）。

### 1.3 自动化测试的主要难点与挑战
* **“编译优化”过程的非确定性与随机搜索收敛性**：自动化测试如何验证优化器在有限步骤内能够确定性提升模型得分。
* **思维链（Rationale）多字段反序列化解析稳定性**：大模型输出带有 `Reasoning:` 与 `Answer:` 的文本时，解析器的容错性。
* **复杂模块参数冻结与持久化**：编译生成的示例（Demos）能否无损保存为 JSON/文件并在生产环境热加载。

---

## 2. 测试架构全景与分层策略

DSPy 建立了以模块化状态机与无模型模拟评估为核心的自动化测试网：

```
┌────────────────────────────────────────────────────────┐
│             Teleprompter Compiler Benchmark Tests      │  编译优化层：验证 BootstrapFewShot 在样本集上的增益
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Pipeline Modules & CoT Integration Suite       │  管道模块层：测试 Predict, ChainOfThought 字段绑定
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Signature Compiler & Prompt Formatter Unit Tests  │  契约语法层：验证 InputField/OutputField 转换为 Prompt
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Deterministic DummyLM & Scoring Harness        │  虚拟桩环境：内存提供确定性响应，驱动编译状态演进
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：声明式 Signature 编译与多字段提取断言

#### 3.1.1 业务功能背景
开发者定义一个复杂的签名：
```python
class TextSummarizer(dspy.Signature):
    """Summarize long text with bullet points."""
    text = dspy.InputField(desc="Raw document")
    summary = dspy.OutputField(desc="Condensed key points")
    confidence = dspy.OutputField(desc="Confidence score 1 to 5")
```
系统必须正确将类定义转换为带有规范格式约束的字符串提示，并在模型返回时精准提取出 `summary` 与 `confidence` 两个独立属性。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 单元测试）**：
```python
import pytest
import dspy
from dspy.utils.dummies import DummyLM

def test_signature_execution_and_field_extraction():
    # 1. 注入 DummyLM 预置输出
    fake_response = """
Reasoning: Let's extract key points.
Summary: AI development is moving towards compiled pipelines.
Confidence: 5
"""
    lm = DummyLM(answers=[fake_response])
    dspy.settings.configure(lm=lm)

    # 2. 组装 ChainOfThought 模块
    module = dspy.ChainOfThought(TextSummarizer)

    # 3. 触发预测
    prediction = module(text="Detailed explanation of DSPy...")

    # 4. 严格字段断言
    assert "AI development is moving" in prediction.summary
    assert prediction.confidence == "5"
    assert hasattr(prediction, "rationale") # 验证思维链中间过程被正确捕获
```

---

### 3.2 功能二：优化器（BootstrapFewShot）编译增益与示范样本注入测试

#### 3.2.1 业务功能背景
未编译的原始模块由于缺少示例，准确率为 0%；使用 `BootstrapFewShot` 编译器在 3 个训练样本上自动编译后，系统自动挖掘出 2 个高质量示例并内嵌进 Prompt。测试断言编译后模块的得分显著提高且 Prompt 中包含了生成的 `Demo`。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
def test_bootstrap_fewshot_optimizer_compilation():
    # 构造极小训练集与打分函数
    trainset = [
        dspy.Example(question="2+2", answer="4").with_inputs("question"),
        dspy.Example(question="3+3", answer="6").with_inputs("question"),
    ]
    metric = lambda example, pred, trace=None: example.answer.strip() == pred.answer.strip()

    # 1. 未编译模块
    uncompiled = dspy.Predict("question -> answer")

    # 2. 启动优化器编译
    teleprompter = dspy.teleprompt.BootstrapFewShot(metric=metric, max_bootstrapped_demos=2)
    compiled_program = teleprompter.compile(student=uncompiled, trainset=trainset)

    # 3. 严格断言：编译生成的模块内部必须成功注入了 Demos
    assert len(compiled_program.predictors()[0].demos) > 0
    # 验证保存与加载机制无损
    compiled_program.save("compiled_math.json")
    loaded_program = dspy.Predict("question -> answer")
    loaded_program.load("compiled_math.json")
    assert len(loaded_program.demos) == len(compiled_program.predictors()[0].demos)
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **DummyLM 确定性词表回放**：
   * 测试套件完全依赖内置的 `DummyLM`，根据传入的 prompt 模式匹配返回预定的回答数组，执行 300+ 单元测试不消耗一分钱 API 费用。
2. **模块序列化兼容性回归**：
   * 自动比对不同版本导出的 JSON 权重文件，确保编译产物向后兼容。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **将 Prompt 视为“可编译与可测试的代码参数”**：告别随意修改的文本提示词，将输入输出规范定义为强类型接口（Signature），在自动化测试中严密断言接口输入输出契约。
* **为 AI 优化过程建立“指标单调递增断言”**：对于具有自动优化、微调、示例优选功能的系统，测试的重心是验证“经过编译/优化后的版本在测试集上的指标不低于原始基线”。
