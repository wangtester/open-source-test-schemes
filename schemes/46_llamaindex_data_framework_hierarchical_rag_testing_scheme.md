# LlamaIndex 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[LlamaIndex](https://github.com/run-llama/llama_index) 是全球最顶级的大语言模型数据接入与上下文增强（RAG）框架。系统基于 Python 构建，提供业界最完备的文档连接器、分层节点切分（Node Parsers）、分层树形索引（Hierarchical / Summary / Vector Index）、**递归检索器（Recursive Retriever）**与知识图谱融合检索（Graph RAG）。

### 1.2 被测系统核心组成
1. **分层节点切分与关系映射器（Hierarchical Node Parser）**：将父文档递归切分多层级父子节点（Parent-Child Nodes），建立双向引用指针（`relationships`）。
2. **递归检索器（Recursive Retriever）**：高精检索小块（Leaf Nodes），定位后沿引用指针向上回溯提取包含完整语义的大块（Parent Node）。
3. **路由查询引擎（Router Query Engine）**：根据用户问题语义，动态选择路由至向量检索、摘要索引或 SQL 结构化引擎。
4. **元数据过滤引擎（Metadata Filtering Engine）**：支持在向量搜索中执行复杂的前置/后置元数据逻辑组合（AND/OR/IN 表达式）。

### 1.3 自动化测试的主要难点与挑战
* **树形节点双向指针在序列化后的引用完整性**：深层节点在保存为 JSON 或存入向量库后指针是否损坏。
* **高维向量距离排序与元数据过滤的协同测试**：如何确保在向量近似搜索的同时，元数据硬性过滤 100% 生效。
* **多步查询流水线（Query Pipeline）的数据流打桩**：在无真实网络与模型下验证检索、重排序与合成的全过程。

---

## 2. 测试架构全景与分层策略

LlamaIndex 建立了以纯数学向量桩与数据结构不变性为核心的自动化测试网：

```
┌────────────────────────────────────────────────────────┐
│             Query Engine & Synthesis Pipeline Tests    │  合成管道层：端到端验证从查询输入到最终回答组装
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Recursive Retriever & Node Pointer Traversal      │  检索遍历层：验证子节点命中后向上回溯父节点的正确性
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Metadata Filter & Logical Expression Tests     │  过滤引擎层：单测验证多条件复合表达式转换为底层 DB 过滤
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         MockEmbedding & Node Serialization Unit Tests  │  向量数据层：内存模拟确定性向量距离，验证节点树结构
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：父子节点递归检索器（Parent-Child Recursive Retriever）回溯断言

#### 3.1.1 业务功能背景
一篇包含 2,000 字的技术规范（Parent Node）被切分为 4 个 500 字的子节点（Child Nodes）。检索时，只有 Child Node 2 匹配了查询关键词；Recursive Retriever 捕获到命中后，自动沿 `parent_node_id` 回溯提取完整的 Parent Node 作为上下文送入大模型，保证推理不丢失全局背景。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 检索器集成测试）**：
```python
import pytest
from llama_index.core.schema import TextNode, NodeRelationship
from llama_index.core.retrievers import RecursiveRetriever

def test_recursive_retriever_traverses_to_parent_node():
    # 1. 构造父节点与子节点
    parent_node = TextNode(text="Full architecture document of the system...", id_="parent_001")
    child_node = TextNode(
        text="Target specific feature paragraph.",
        id_="child_001",
        relationships={NodeRelationship.PARENT: parent_node.as_related_node_info()}
    )

    # 2. 将子节点存入向量索引，父节点存入文档存储（Docstore）
    index = create_test_index([child_node])
    docstore = {parent_node.id_: parent_node}

    # 3. 构造递归检索器
    retriever = RecursiveRetriever(
        "vector",
        retriever_dict={"vector": index.as_retriever()},
        node_dict=docstore
    )

    # 4. 触发检索
    nodes = retriever.retrieve("feature query")

    # 5. 严格断言：最终返回的必须是被解析回溯的父节点，而非子节点短文本
    assert len(nodes) == 1
    assert nodes[0].node.id_ == "parent_001"
    assert "Full architecture document" in nodes[0].node.text
```

---

### 3.2 功能二：元数据硬性过滤（Metadata Filtering）测试

#### 3.2.1 业务功能背景
知识库包含两篇文档：Doc A (`category="finance", year=2024`)，Doc B (`category="tech", year=2024`)。用户查询指定过滤条件 `category == "finance"`。系统必须在向量距离计算前后严格剔除 Doc B，即便 Doc B 的文本语义相似度更高。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
from llama_index.core.vector_stores import MetadataFilters, MetadataFilter

def test_metadata_filtering_strictly_excludes_mismatched_documents():
    filters = MetadataFilters(filters=[MetadataFilter(key="category", value="finance")])
    
    # 构造测试检索器
    retriever = index.as_retriever(similarity_top_k=2, filters=filters)
    results = retriever.retrieve("any general query")

    # 断言：返回结果中所有文档的 category 属性必须 100% 等于 finance
    for res in results:
        assert res.node.metadata["category"] == "finance"
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **MockEmbedding 确定性正交向量库**：
   * 测试套件自研 `MockEmbedding`，根据字符串长度或哈希生成确定性的正交向量，使得测试用例中的余弦相似度完全受控可预测。
2. **纯内存 VectorStore 夹具**：
   * 使用 SimpleVectorStore（纯 Python 字典实现），无需部署 Qdrant/Pinecone 等外部服务即可覆盖所有索引与检索逻辑。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **复杂 RAG 必须断言“上下文拓扑还原（Context Topological Recovery）”**：小块高精匹配、大块完整理解是现代 RAG 的标配；自动化测试必须验证指针回溯机制能够准确复原全局上下文。
* **向量测试必须排除“外部模型漂移”**：在测试检索器逻辑时，严禁调用在线 Embedding API；使用确定性的数学 Mock 向量生成器，才能真正测出索引与过滤算法的正确性。
