# Dify 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Dify](https://github.com/langgenius/dify) 是全球领先的开源 LLM 应用开发与可视化工作流编排平台。系统在工程落地中高度依赖 AI 辅助编码与智能体流程。前端基于 Next.js + React Flow 构建强大的可视化拖拽画布，后端基于 Python (Flask / Celery) + PostgreSQL + Redis 构建高吞吐异步工作流与知识库 RAG 管道。

### 1.2 被测系统核心组成
1. **DAG 工作流编排引擎（Workflow Graph Engine）**：对画布节点（LLM、代码、知识库检索、条件分支、HTTP 请求、迭代器）进行拓扑排序与异步调度执行。
2. **知识库文档清洗管道（Dataset Ingestion Pipeline）**：PDF/Word 文本提取、多策略分块（Chunking）、向量化嵌入与混合检索（Hybrid Search）。
3. **DSL 导入导出与版本控制（Workflow DSL Engine）**：工作流配置的双向 YAML 序列化与架构校验。
4. **前端 React Flow 交互画布**：节点拖拽吸附、端口连线校验、运行抽屉（Run Panel）变量输入与实时状态点亮。

### 1.3 自动化测试的主要难点与挑战
* **有向无环图（DAG）复杂拓扑与条件分支执行验证**：分支条件、多前驱合并、循环迭代节点的执行时序与数据流转极其复杂。
* **文件解析与 RAG 文本切分的边界情况**：各种破损 PDF、超大表格、多级标题在文档切片中的表现。
* **工作流异步长轮询测试稳定性**：大型工作流耗时数十秒，自动化测试必须兼顾速度与断言可观测性。

---

## 2. 测试架构全景与分层策略

Dify 建立了涵盖工作流引擎、知识库清洗与画布 E2E 的全栈测试体系：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Workflow Canvas E2E             │  画布交互层：驱动 React Flow 节点拖拽、连线与试运行
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Workflow Graph Engine & Node Integration Tests    │  流程引擎层：构建多节点 DAG 图，断言数据沿边传递与拓扑调度
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Dataset Ingestion & Chunking Unit Tests        │  知识库管道层：验证文本切片算法、召回率、重排序模型
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│         Workflow DSL YAML Schema Validation Suite      │  契约语法层：比对导入导出 YAML DSL 的一致性与向下兼容
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：DAG 工作流拓扑调度与条件分支执行断言

#### 3.1.1 业务功能背景
用户编排了一个工作流：`Start -> If-Else (Branch 1 & 2) -> LLM Node -> End`。当输入变量满足条件时，仅触发 `Branch 1` 链路，`Branch 2` 节点被标记为 `SKIPPED`，最终聚合节点成功接收数据并输出。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 工作流调度引擎测试）**：
```python
import pytest
from core.workflow.graph_engine import GraphEngine
from core.workflow.nodes import StartNode, IfElseNode, LLMNode, EndNode

def test_workflow_conditional_branching_execution():
    # 1. 编排测试 DAG 图数据结构
    graph_config = {
        "nodes": [
            {"id": "start", "type": "start", "data": {"variables": [{"name": "score", "type": "number"}]}},
            {"id": "if_else", "type": "if-else", "data": {"conditions": [{"variable": "start.score", "operator": ">=", "value": 60}]}},
            {"id": "pass_llm", "type": "llm", "data": {"prompt": "Passed with {{start.score}}"}},
            {"id": "fail_llm", "type": "llm", "data": {"prompt": "Failed with {{start.score}}"}},
            {"id": "end", "type": "end", "data": {"outputs": [{"value_selector": ["pass_llm", "result"]}]}}
        ],
        "edges": [
            {"source": "start", "target": "if_else"},
            {"source": "if_else", "target": "pass_llm", "sourceHandle": "true"},
            {"source": "if_else", "target": "fail_llm", "sourceHandle": "false"},
            {"source": "pass_llm", "target": "end"}
        ]
    }

    # 2. 传入变量 score = 85 执行引擎
    engine = GraphEngine(config=graph_config)
    result = engine.run(inputs={"score": 85})

    # 3. 结构化断言：验证节点状态与分支路径
    assert result.status == "succeeded"
    assert engine.get_node_status("pass_llm") == "SUCCEEDED"
    assert engine.get_node_status("fail_llm") == "SKIPPED" # 验证未走的分支被正确跳过
```

---

### 3.2 功能二：知识库长文本分块（Chunking）与重叠字数边界测试

#### 3.2.1 业务功能背景
用户上传一份 Markdown 文档，设置分块最大长度为 500 Token，重叠长度（Overlap）为 50 Token。清洗管道必须严格保证分块不割裂完整的段落代码块，且重叠字符精确对齐。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
def test_markdown_splitter_preserves_code_blocks():
    doc_content = "# Title\n\nSome text.\n\n```python\ndef calculate():\n    return 42\n```\n\nMore text."
    splitter = MarkdownTextSplitter(chunk_size=50, chunk_overlap=10)
    
    chunks = splitter.split_text(doc_content)
    
    # 断言代码块未被截断在中间
    code_chunk = next(c for c in chunks if "def calculate():" in c)
    assert "return 42" in code_chunk
    assert "```" in code_chunk
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **工作流 DSL 快照全回归**：
   * 在代码仓库维护上百个代表典型业务流的 `workflow_golden.yml`。每次引擎升级，自动载入所有 DSL 并执行静态拓扑分析与模拟执行。
2. **Celery 异步任务测试模式**：
   * 测试环境中配置 `CELERY_TASK_ALWAYS_EAGER = True`，让异步任务退化为同步直调，确保测试秒级反馈。

---

## 5. 对复杂系统的实战借鉴意义

* **流程编排系统必须具备“图执行追踪（Graph Tracer）”断言能力**：测试工作流时不能仅断言最终输出，必须断言图的每个节点状态（SUCCEEDED, FAILED, SKIPPED），确保分支决策绝对符合预期。
* **将文本切分做成纯数学与规则覆盖测试**：知识库等数据管道的预处理层，收集各种畸形文件进行单测断言，比等到向量检索阶段再去调试要容易百倍。
