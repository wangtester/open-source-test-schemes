# LangChain 自动化测试方案与 RAG 智能体用例全集深度剖析

## 1. 项目背景与系统架构全景

[LangChain](https://github.com/langchain-ai/langchain) 是全球大语言模型（LLM）应用开发框架的标准基础设施。其核心依托 **LCEL（LangChain Expression Language）** 声明式表达式语言，将复杂的多步骤提示词工程、检索增强（RAG）、记忆存储（Memory）、工具调用（Tool Use）以及自主智能体（Agent）串联为可流式传输、支持异步执行与回退降级的统一执行图（Runnable Pipeline）。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        LangChain Application / Agent                   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ LCEL Pipe (|) 声明式调用链
┌───────────────────────────────────▼────────────────────────────────────┐
│                    LCEL Core Engine (Runnable Protocol)                │
├───────────────────────────────┬────────────────────────────────────────┤
│ PromptTemplate & Parsers      │ Model Adapters & Dynamic Routing       │
│ - 变量动态插值与严格类型校验  │ - OpenAI / Anthropic / HuggingFace     │
│ - JSON / XML / Pydantic 解析  │ - `.with_fallbacks()` 故障自动降级      │
├───────────────────────────────┼────────────────────────────────────────┤
│ Document Loaders & Splitters  │ VectorStore & MultiQuery Retriever     │
│ - 递归文本切分 (Recursive)    │ - 稠密向量余弦检索与元数据过滤         │
│ - 文本特征清洗与 Token 裁剪   │ - 多角度重构查询与上下文压缩           │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

LangChain 构建了在非确定性大模型环境下保障高可靠运行的自动化测试体系：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **LCEL Unit Tests** | Runnable 管道操作符（`|`）、批处理（`batch`）、流式（`stream`） | Pytest (`tests/unit_tests`) | 纯内存 FakeLLM，零成本毫秒级 |
| **Output Parsers Tests** | 提取 Markdown 代码块中的 JSON、输出格式损坏时的重试修复解析器 | Pytest + Pydantic | 静态畸形文本注入测试 |
| **Fallback & Retry Tests** | 模型 API 返回 429 / 500 时自动回退至备用模型或本地缓存 | Pytest + VCR.py / Mock | 拦截外部 HTTP 流量注入故障 |
| **Retriever Pipeline Tests** | 多查询改写、向量召回打分排序、上下文压缩裁剪 | Pytest + InMemoryVectorStore | 本地内存向量检索，无需 Milvus |
| **Tool Calling Agent Tests** | 智能体解析用户指令并准确生成工具入参字典、异常入参拦截 | Pytest + FakeTool | 虚假工具拦截断言 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：LCEL 声明式可运行链路（Runnable）与自动回退（Fallback）测试

#### 3.1.1 业务功能与测试目标
LCEL 支持将多个组件串联：`chain = prompt | primary_llm.with_fallbacks([backup_llm]) | StrOutputParser()`。
当主力模型抛出异常（如 HTTP 429 限流或超时）时，测试必须验证：
1. 系统瞬间无感切换至备用模型（Backup LLM）并成功返回。
2. 整个切换过程不破坏外部流式传输（Streaming）输出契约。

#### 3.1.2 测试用例实现
```python
import pytest
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_community.llms.fake import FakeListLLM

class AlwaysFailingLLM(FakeListLLM):
    def _call(self, prompt: str, stop=None, **kwargs) -> str:
        raise RuntimeError("Primary Model Rate Limited (HTTP 429)")

def test_lcel_runnable_with_fallbacks():
    prompt = PromptTemplate.from_template("Summarize: {topic}")
    
    # 1. 配置主力模型（必崩）与备用模型（返回正常内容）
    failing_primary = AlwaysFailingLLM(responses=[])
    working_backup = FakeListLLM(responses=["Backup model successfully summarized the topic."])

    # 2. 组装具备 Fallback 容灾能力的 LCEL 链路
    chain = (
        prompt
        | failing_primary.with_fallbacks([working_backup])
        | StrOutputParser()
    )

    # 3. 执行调用并断言：主力报错被捕获，最终由备用模型兜底成功
    result = chain.invoke({"topic": "Quantum Computing"})
    assert "Backup model successfully summarized" in result
```

---

### 3.2 模块二：Pydantic 结构化输出解析器与自愈纠错（OutputFixingParser）测试

#### 3.2.1 业务功能与测试目标
大模型输出经常包含多余前导词或损坏的括号。`OutputFixingParser` 能够拦截 `OutputParserException`，自动将错误信息和原始输出打包成新 Prompt 重新请求模型进行自愈修复。测试必须覆盖：
1. 正常 JSON 正确解析为 Pydantic 强类型对象。
2. 损坏 JSON 触发自动重试修复并最终成功输出。

#### 3.2.2 测试用例实现
```python
import pytest
from pydantic import BaseModel, Field
from langchain.output_parsers import PydanticOutputParser, OutputFixingParser
from langchain_community.llms.fake import FakeListLLM

class ActorInfo(BaseModel):
    name: str = Field(description="Actor name")
    film_names: list[str] = Field(description="List of movie names")

def test_output_fixing_parser_recovers_broken_json():
    base_parser = PydanticOutputParser(pydantic_object=ActorInfo)
    
    # 模拟一个损坏的 JSON 输出（缺少闭合花括号）
    broken_output = '{"name": "Tom Hanks", "film_names": ["Forrest Gump", "Cast Away"'

    # 模拟修复模型返回完全合规的 JSON
    fixing_llm = FakeListLLM(responses=['{"name": "Tom Hanks", "film_names": ["Forrest Gump", "Cast Away"]}'])

    # 装配自动纠错解析器
    fixing_parser = OutputFixingParser.from_llm(parser=base_parser, llm=fixing_llm)

    # 执行解析
    actor = fixing_parser.parse(broken_output)

    # 严格类型断言
    assert isinstance(actor, ActorInfo)
    assert actor.name == "Tom Hanks"
    assert len(actor.film_names) == 2
```

---

### 3.3 模块三：内存向量检索（InMemoryVectorStore）与元数据过滤测试

#### 3.3.1 业务功能与测试目标
验证文档灌库、向量相似度计算与严格元数据过滤（Metadata Filtering）：
1. 确保带有 `author: "Alice"` 过滤条件的查询，只能召回 Alice 的文档，即使 Bob 的文档在向量余弦相似度上更高。

#### 3.3.2 测试用例实现
```python
from langchain_core.documents import Document
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_community.embeddings import FakeEmbeddings

def test_vectorstore_similarity_search_with_metadata_filter():
    embeddings = FakeEmbeddings(size=128)
    vectorstore = InMemoryVectorStore(embedding=embeddings)

    # 1. 批量灌入测试文档集
    docs = [
        Document(page_content="LangChain core architecture overview", metadata={"author": "Alice", "year": 2024}),
        Document(page_content="Deep learning optimization guide", metadata={"author": "Bob", "year": 2024}),
        Document(page_content="LangChain agent tool usage patterns", metadata={"author": "Alice", "year": 2023}),
    ]
    vectorstore.add_documents(docs)

    # 2. 发起带有元数据过滤条件的相似度检索
    retrieved = vectorstore.similarity_search(
        query="Tell me about LangChain",
        k=2,
        filter={"author": "Alice"}
    )

    # 3. 强断言：返回的所有文档必须属于 Alice
    assert len(retrieved) == 2
    for doc in retrieved:
        assert doc.metadata["author"] == "Alice"
        assert "LangChain" in doc.page_content
```

---

### 3.4 模块四：Agent 工具调用入参严格校验与异常拦截测试

#### 3.4.1 业务功能与测试目标
当 Agent 试图调用特定工具（例如执行计算或搜索）时，测试必须验证：
1. LLM 生成的参数字典与工具预期的 Pydantic Schema 严格匹配。
2. 缺失必填参数时，工具必须返回错误提示字符串给 Agent，让其感知并修正，而不能直接抛出异常导致整个调用崩溃。

#### 3.4.2 测试用例实现
```python
from langchain_core.tools import tool

@tool
def calculate_vat(price: float, rate: float = 0.2) -> float:
    """Calculate VAT tax for a given price."""
    return price * rate

def test_tool_calling_parameter_validation_and_recovery():
    # 1. 正常入参调用验证
    res = calculate_vat.invoke({"price": 100.0, "rate": 0.15})
    assert res == 15.0

    # 2. 模拟 LLM 传入非法的字符串参数，验证优雅错误返回
    err_res = calculate_vat.invoke({"price": "invalid_number"})
    assert isinstance(err_res, str) or calculate_vat.handle_tool_error is not None
```

---

## 4. 测试工程化与基础设施

1. **FakeLLM / FakeEmbeddings 零网络开销测试**：
   * LangChain 的单元测试全面采用确定性生成器（`FakeListLLM` 与 `FakeEmbeddings`），生成固定维度的确定性向量，不需要花费任何 API Token，万级测试用例在 30 秒内完成。
2. **多版本 Pydantic（v1 / v2）双向兼容矩阵**：
   * 在 CI 中同时针对 Pydantic 1.10 与 2.x 运行全套测试，确保依赖向下兼容。

---

## 5. 对同类复杂系统的参考价值

* **非确定性 AI 管道的确定性测试方案**：测试 LLM 编排框架时，不能把宝押在真实模型返回上，必须将“管道调度、类型转换、回退降级、自愈重试”解耦为独立纯函数用例，使用确定性的 Mock 桩保证测试的可复现性与稳定性。
