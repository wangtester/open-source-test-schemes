# Instructor 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Instructor](https://github.com/jxnl/instructor) 是全球最主流的基于 Pydantic 的大语言模型类型安全与结构化输出（Structured Outputs）框架。系统基于 Python 构建，能够将任意复杂的 Pydantic 数据模型无缝转换为各种大模型提供商的 Function Calling / Tool Calling JSON Schema，并创新实现了 **“自动验证与自愈重试（Reask on Validation Error）”** 循环。

### 1.2 被测系统核心组成
1. **模式提取与 Schema 编译器（Schema Compiler）**：将 Pydantic 类的字段、注释（Docstrings）、验证器（Validators）自动编译为符合 OpenAI / Anthropic 规范的 JSON Schema。
2. **校验自愈重试循环（Validation & Reask Engine）**：当模型返回的数据未通过 Pydantic 校验规则时，自动捕获 `ValidationError`，将错误原因格式化为反馈消息重新喂给模型，直至数据合法或达到 `max_retries`。
3. **流式增量部分解析器（Partial Streaming Parser）**：在流式生成过程中，动态将残缺不全的 JSON 文本智能补全并反序列化为具有部分字段的合法 Pydantic 实例。
4. **多模型补丁装配器（Client Patches）**：为 OpenAI, Anthropic, Gemini, Groq, Ollama 等官方 SDK 打上无侵入包装。

### 1.3 自动化测试的主要难点与挑战
* **Pydantic 验证器失败后自愈链路的确定性模拟**：如何在单测中精准模拟“第 1 次返回非法数据、第 2 次在提示词指导下返回合法数据”的交互状态。
* **流式 JSON 动态补全状态机测试**：模型每输出几个字符，局部 JSON 树都在演化，测试需验证任意截断时刻反序列化均不崩溃。
* **Pydantic v1 与 v2 双内核架构的兼容性**：核心元数据 API 在两个大版本间的行为一致性保证。

---

## 2. 测试架构全景与分层策略

Instructor 建立了以受控模型桩和流式切片状态机为核心的测试矩阵：

```
┌────────────────────────────────────────────────────────┐
│             Multi-Provider Client Patch E2E Tests      │  提供商适配层：针对 OpenAI/Anthropic SDK 执行包装打桩
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Validation & Reask Loop Integration Suite      │  自愈重试层：验证捕获 ValidationError 并自动重问机制
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Partial Streaming Incremental Json Tests       │  流式解析层：逐字符灌入半残 JSON，断言局部对象合法性
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Pydantic Schema Compiler Unit Tests            │  模式编译层：单测验证嵌套类、Union、Enum 生成的 Schema
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：Pydantic 校验失败自动反思重试（Reask Loop）测试

#### 3.1.1 业务功能背景
定义数据模型 `UserProfile`，其中字段 `age` 包含验证器 `@field_validator('age')` 限制必须大于等于 18。模型第 1 次输出了 `{"name": "Bob", "age": 15}` 导致校验失败。系统自动将报错 "age must be >= 18" 注入上下文重问，模型第 2 次修正为 `{"name": "Bob", "age": 20}`。系统最终成功返回合法的 `UserProfile` 实例。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 状态自愈测试）**：
```python
import pytest
from pydantic import BaseModel, field_validator
import instructor
from unittest.mock import MagicMock

class UserProfile(BaseModel):
    name: str
    age: int

    @field_validator('age')
    @classmethod
    def validate_age(cls, v):
        if v < 18:
            raise ValueError("Age must be at least 18 years old")
        return v

def test_instructor_reask_recovers_from_validation_error():
    # 1. 构造 Mock 客户端：第 1 次返回非法 age=15，第 2 次返回合法 age=20
    mock_client = MagicMock()
    mock_client.chat.completions.create.side_effect = [
        make_mock_tool_call_response('{"name": "Bob", "age": 15}'),
        make_mock_tool_call_response('{"name": "Bob", "age": 20}')
    ]

    # 2. 装配 Instructor 客户端
    client = instructor.from_openai(mock_client)

    # 3. 发起调用，设置 max_retries=2
    user = client.chat.completions.create(
        model="gpt-4o",
        response_model=UserProfile,
        messages=[{"role": "user", "content": "Extract Bob age 20"}],
        max_retries=2
    )

    # 4. 严格断言
    assert user.name == "Bob"
    assert user.age == 20
    # 验证底层真实发起了 2 次对话重试
    assert mock_client.chat.completions.create.call_count == 2
    # 验证第 2 次请求的消息体中包含了 ValidationError 错误回执
    second_call_messages = mock_client.chat.completions.create.call_args_list[1].kwargs["messages"]
    assert any("Age must be at least 18" in str(m) for m in second_call_messages)
```

---

### 3.2 功能二：流式局部 JSON（Partial Streaming）渐进解析断言

#### 3.2.1 业务功能背景
模型正在流式输出 `{"name": "Alice", "tags": ["admin", "dev"`，此时大括号与数组尚未闭合。前端希望实时读取到已经成型的属性。增量解析器必须将其安全补全为 `{"name": "Alice", "tags": ["admin"]}` 并反序列化。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
from instructor.dsl.partial import Partial

def test_partial_streaming_incremental_yield():
    chunks = [
        '{"name": "Al',
        'ice", "age": 3',
        '0}'
    ]
    partial_model = Partial[UserProfile]
    
    # 模拟流式增量灌入并断言每次 yield 的对象
    # 第 1 块：只有 name='Al'
    obj1 = partial_model.from_raw_json(chunks[0])
    assert obj1.name == "Al"
    assert obj1.age is None

    # 第 3 块全部到位：完整对象
    obj3 = partial_model.from_raw_json("".join(chunks))
    assert obj3.name == "Alice"
    assert obj3.age == 30
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Pydantic v1 / v2 矩阵双轨流水线**：
   * 在 GitHub Actions 中配置两套虚拟环境，分别安装 `pydantic<2.0` 与 `pydantic>=2.0`，全量执行相同的测试集，验证核心接口的绝对向前兼容。
2. **纯内存无 Key 零网络门禁**：
   * 编写专用夹具拦截所有的网络通信，无论是否有配置真实 API 密钥，本地与 CI 流水线均能在 3 秒内执行完几百个结构化提取用例。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **将“数据校验失败”设计为可恢复的闭环测试**：对于依赖 LLM 输出结构化数据的业务，单测中必须故意注入字段缺失、枚举值越界、业务逻辑矛盾，断言自愈重问机制能够正确提取错误堆栈并成功修复。
* **流式前端绑定的核心是“半残 JSON 容错断言”**：构建高保真打字机 UI 时，把数据在半个单词处切断并验证解析器行为，是防止前端报错白屏的最佳工程实践。
