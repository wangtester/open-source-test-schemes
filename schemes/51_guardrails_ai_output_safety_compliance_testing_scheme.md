# Guardrails AI 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Guardrails AI](https://github.com/guardrails-ai/guardrails) 是全球企业级应用最广泛的大语言模型输入输出安全合规、结构化验证与风险控制护栏框架。系统基于 Python 构建，提供强大的 **`Guard` 运行时守卫**，内置数十种标准化校验器（Validators），支持在模型输出违规时自动执行**屏蔽（Filter）、拒绝（Refrain）、自动修复（Fix）或反思重问（ReAsk）**。

### 1.2 被测系统核心组成
1. **多策略失败响应引擎（On-Fail Policy Engine）**：定义当校验器（Validator）失败时采取的行为模式（`noop`, `filter`, `refrain`, `fix`, `reask`, `exception`）。
2. **PII 隐私嗅探与自动掩码管道（PII Scrubbing & Anonymization）**：基于正则与 NLP 命名实体识别技术，检测电话、身份证、信用卡、个人姓名并执行脱敏。
3. **结构化 Schema 验证器网格（Structured Schema Validators）**：涵盖 Valid Range, Valid URL, SQL Syntax, Toxicity, Hallucination 等维度的安全网格。
4. **ReAsk 协议自动构造器（ReAsk Protocol Builder）**：将验证失败的字段与修正指令自动化装配为增量 Prompt 进行自愈修正。

### 1.3 自动化测试的主要难点与挑战
* **多种 On-Fail 处置策略的状态机分支覆盖**：同一个校验规则在不同策略下的返回形态完全不同（从直接抛错到无损剔除）。
* **PII 敏感信息的误杀（False Positive）与漏杀（False Negative）边界**：各种格式畸形的手机号或国际电话的测试样本覆盖。
* **嵌套结构化字段中局部失败的精准嫁接**：一个包含 10 个字段的 JSON，仅 1 个字段触发 ReAsk 时，如何局部替换而不损坏其它 9 个字段。

---

## 2. 测试架构全景与分层策略

Guardrails AI 建立了以规则校验器单测与多策略管道集成为核心的测试网：

```
┌────────────────────────────────────────────────────────┐
│             Guard Full-Pipeline ReAsk Integration     │  自愈全流程：验证模型生成 -> 字段报错 -> ReAsk 修正成功
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      6 On-Fail Policies Matrix Behavioral Tests        │  策略矩阵层：单测覆盖 filter, refrain, fix, reask, exception
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         PII Anonymization & Security Validator Suite   │  安全校验层：单测验证脱敏算法、SQL 语法与毒性分类器
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Schema Parser & AST Validation Unit Tests      │  模式解析层：单测验证 .rail 声明式规范与 Pydantic 编译
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：On-Fail 策略矩阵（Fix vs ReAsk vs Filter）行为断言

#### 3.1.1 业务功能背景
字段限制为浮点数范围 `0.0 <= score <= 1.0`。模型输出了非法值 `1.5`：
* 当配置 `on_fail="fix"` 时，系统自动截断并修正为 `1.0`；
* 当配置 `on_fail="filter"` 时，系统将该字段从输出字典中安全剔除；
* 当配置 `on_fail="reask"` 时，系统自动向模型发起下一轮修正对话。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 策略参数化矩阵测试）**：
```python
import pytest
from guardrails import Guard
from guardrails.hub import ValidRange
from pydantic import BaseModel, Field

def test_on_fail_fix_clamps_value():
    class OutputModel(BaseModel):
        score: float = Field(validators=[ValidRange(min=0.0, max=1.0, on_fail="fix")])

    guard = Guard.from_pydantic(output_class=OutputModel)
    # 模拟模型输出超出范围的非法值
    validated_output = guard.parse('{"score": 1.5}')
    
    # 严格断言：自动修正为上限 1.0
    assert validated_output.validated_output["score"] == 1.0

def test_on_fail_filter_removes_invalid_field():
    class OutputModel(BaseModel):
        score: float = Field(validators=[ValidRange(min=0.0, max=1.0, on_fail="filter")])

    guard = Guard.from_pydantic(output_class=OutputModel)
    validated_output = guard.parse('{"score": 1.5}')
    
    # 严格断言：字段被物理过滤剔除
    assert "score" not in validated_output.validated_output
```

---

### 3.2 功能二：PII 敏感隐私信息识别与掩码脱敏断言

#### 3.2.1 业务功能背景
模型输出："Contact customer John Doe at john@acme.com or call 555-123-4567."。开启隐私护栏后，文本必须被清洗为："Contact customer `<NAME>` at `<EMAIL>` or call `<PHONE_NUMBER>`."，且原始敏感明文绝对不进入下游存储。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
from guardrails.hub import DetectPII

def test_pii_anonymizer_masks_all_identifiable_attributes():
    validator = DetectPII(pii_entities=["EMAIL_ADDRESS", "PHONE_NUMBER"], on_fail="fix")
    
    dirty_text = "Send credentials to admin@company.org or text 123-456-7890 immediately."
    clean_text = validator.validate(dirty_text)

    # 严格断言：原始邮箱与电话被完全遮蔽
    assert "admin@company.org" not in clean_text
    assert "123-456-7890" not in clean_text
    assert "<EMAIL_ADDRESS>" in clean_text
    assert "<PHONE_NUMBER>" in clean_text
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **零大模型调用的规则级纯单测（Offline Rule Engine）**：
   * 95% 的校验器基于纯正则表达式、语法树词法分析或本地微型分类器构建，测试套件无需任何外部网络连接即可在毫秒内完成数百个规则验证。
2. **ReAsk 递归深度熔断保护**：
   * 在涉及 ReAsk 自愈的端到端集成测试中，设置 `max_reasks=2`，强制断言系统在模型持续输出非法数据时能够平滑抛出受管异常并终止，杜绝无限死循环。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **将合规与安全防护抽离为“管道断言拦截器”**：对于面向公众或金融风控的 AI 系统，安全检测不能零散写在业务代码中，必须建立专门的护栏组件，并针对 PII 泄露、SQL 注入编写高密度的负向用例集。
* **为业务数据建立明确的“降级处理策略矩阵”**：当模型输出违规时，系统应该怎么做？在设计测试时，显式验证 `filter`, `fix`, `reask` 等不同场景策略的输出行为，是系统进入高可用状态的关键保障。
