# Dify 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[Dify](https://github.com/langgenius/dify) 是开源的 LLM 应用开发与可视化工作流编排平台。其核心能力涵盖可视化 DAG 工作流编排、企业级 RAG 知识库检索增强、Agent 自主工具调用以及全生命周期 Prompt 工程。

```
┌────────────────────────────────────────────────────────────────────────┐
│             Web UI (Next.js + React Flow 可视化工作流画布)             │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ REST API / WebSocket
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Dify Core Backend (Flask / Celery)                  │
├───────────────────────────────┬────────────────────────────────────────┤
│ Workflow Graph Engine         │ Dataset & RAG Ingestion Pipeline       │
│ - 拓扑排序调度                │ - 多格式文档清洗 (PDF/Docx/HTML)       │
│ - 条件分支与多路并行合并       │ - 语义分块 (Chunking) & 智能清洗       │
│ - 循环迭代器 (Iteration Node) │ - 向量化 + 全文 (BM25) 混合检索        │
├───────────────────────────────┼────────────────────────────────────────┤
│ Plugin & Tool Runtime         │ Workflow DSL Engine                    │
│ - 开放 API 工具沙箱调用       │ - YAML 双向导入导出与版本兼容          │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Dify 构建了覆盖算法逻辑、分布式异步任务与复杂图形界面的完整测试矩阵：

| 测试层级 | 被测模块与用例范围 | 技术选型与运行环境 | 隔离机制与测试策略 |
| :--- | :--- | :--- | :--- |
| **Workflow Engine Tests** | DAG 图拓扑排序、条件分支（If-Else）、迭代器循环、变量抽取器 | Pytest (`core/workflow`) | 内存构造图拓扑，Mock 外部 LLM/API 节点 |
| **RAG Ingestion Tests** | 文本解析器、分块算法、重叠截断、BM25 权重倒排、向量余弦相似度 | Pytest + NumPy | 本地测试文档集，隔离真实向量数据库 |
| **DSL Schema & Migration** | YAML 工作流配置解析、历史版本向上兼容、非法节点连接检测 | Pydantic / PyYAML | 静态模式校验与快照比对 |
| **Tool & Plugin Sandbox** | 第三方 API 认证、参数注入安全校验、超时限制 | Pytest + responses / respx | HTTP 流量拦截与沙箱安全测试 |
| **Celery Tasks Tests** | 知识库异步分块入库、批量运行任务状态回写 | Pytest (`CELERY_TASK_ALWAYS_EAGER`) | 将分布式队列转为内存同步直调 |
| **Canvas E2E Tests** | React Flow 拖拽、连线合法性、运行抽屉变量注入、执行高亮 | Playwright (TypeScript) | 受控 Docker Compose 容器集群 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：DAG 工作流图执行引擎与复杂分支控制测试

#### 3.1.1 业务功能与测试目标
工作流引擎驱动由各种节点构成的有向无环图。测试必须保证：
1. **拓扑排序与依赖执行**：节点严格按照上游依赖顺序触发。
2. **条件分支隔离**：未命中的分支节点状态严格标记为 `SKIPPED`，已命中的分支正常流转。
3. **多前驱聚合（Merge Node）**：聚合节点在上游所有活动前驱节点完成后才开始执行。

#### 3.1.2 测试用例实现
```python
import pytest
from core.workflow.graph_engine import GraphEngine
from core.workflow.entities.variable_pool import VariablePool
from core.workflow.enums import WorkflowNodeExecutionStatus

def test_workflow_conditional_branching_and_skipping():
    # 构造 DAG: Start -> If-Else -> (True: Node A -> Node C) / (False: Node B) -> End
    graph_config = {
        "nodes": [
            {"id": "node_start", "type": "start", "data": {"variables": [{"name": "input_val", "type": "number"}]}},
            {
                "id": "node_condition",
                "type": "if-else",
                "data": {
                    "logical_operator": "and",
                    "conditions": [{"variable_selector": ["node_start", "input_val"], "comparison_operator": ">", "value": 10}]
                }
            },
            {"id": "node_a", "type": "code", "data": {"code": "def main(x):\n    return {'result': x * 2}"}},
            {"id": "node_b", "type": "code", "data": {"code": "def main(x):\n    return {'result': x * 0}"}},
            {"id": "node_end", "type": "end", "data": {"outputs": [{"value_selector": ["node_a", "result"]}]}}
        ],
        "edges": [
            {"source": "node_start", "target": "node_condition"},
            {"source": "node_condition", "target": "node_a", "sourceHandle": "true"},
            {"source": "node_condition", "target": "node_b", "sourceHandle": "false"},
            {"source": "node_a", "target": "node_end"}
        ]
    }

    # 1. 执行用例：输入 input_val = 20，命中 True 分支
    engine = GraphEngine(workflow_config=graph_config)
    execution_result = engine.run(inputs={"input_val": 20})

    # 2. 状态断言
    assert execution_result.status == "succeeded"
    assert engine.get_node_status("node_a") == WorkflowNodeExecutionStatus.SUCCEEDED
    assert engine.get_node_status("node_b") == WorkflowNodeExecutionStatus.SKIPPED  # 关键断言：未走分支必跳过
    assert engine.variable_pool.get(["node_a", "result"]) == 40

    # 3. 执行用例：输入 input_val = 5，命中 False 分支
    engine_false = GraphEngine(workflow_config=graph_config)
    execution_result_false = engine_false.run(inputs={"input_val": 5})

    assert engine_false.get_node_status("node_a") == WorkflowNodeExecutionStatus.SKIPPED
    assert engine_false.get_node_status("node_b") == WorkflowNodeExecutionStatus.SUCCEEDED
```

---

### 3.2 模块二：循环迭代器节点（Iteration Node）边界与中断测试

#### 3.2.1 业务功能与测试目标
迭代器节点负责循环处理数组（如批量总结长列表文档）。测试必须覆盖：
1. 空数组、单元素与多元素遍历。
2. 循环中单个批次异常时的容错策略（继续或终止）。
3. 循环内部变量在外部的聚合输出格式。

#### 3.2.2 测试用例实现
```python
def test_iteration_node_batch_processing_and_error_handling():
    items = ["Apple", "Banana", "Cherry"]
    iteration_config = {
        "iterator_selector": ["node_start", "fruits"],
        "sub_nodes": [
            {"id": "iter_upper", "type": "code", "data": {"code": "def main(fruit):\n    return {'upper': fruit.upper()}"}}
        ],
        "output_selector": ["iter_upper", "upper"],
        "error_handle_mode": "continue_on_error"
    }

    engine = GraphEngine(iteration_config=iteration_config)
    result = engine.run_iteration(input_list=items)

    assert result.status == "succeeded"
    assert result.aggregated_output == ["APPLE", "BANANA", "CHERRY"]
    assert len(result.iteration_traces) == 3
```

---

### 3.3 模块三：知识库文档清洗、语义切分（Chunking）与重叠边界测试

#### 3.3.1 业务功能与测试目标
知识库处理必须将 Markdown、TXT、PDF 转化为特定 Token 上限的切片。测试必须保证：
1. 切分不能打断多行 Markdown 代码块与表格结构。
2. `chunk_overlap` 重叠字符必须精确匹配，避免语境丢失。

#### 3.3.2 测试用例实现
```python
import pytest
from core.rag.cleaner import DocumentCleaner
from core.rag.splitter import MarkdownTextSplitter

def test_markdown_chunking_maintains_tables_and_codeblocks():
    raw_markdown = """
# Section 1
Here is a table:
| Name | Role | Score |
| :--- | :--- | :--- |
| Alice | Admin | 95 |
| Bob | User | 82 |

```python
def critical_logic():
    print("Do not split me!")
```
End of text.
"""
    cleaner = DocumentCleaner()
    cleaned = cleaner.clean(raw_markdown)

    splitter = MarkdownTextSplitter(chunk_size=120, chunk_overlap=20)
    chunks = splitter.split_text(cleaned)

    # 1. 断言分块数量合理
    assert len(chunks) >= 2

    # 2. 核心断言：代码块不允许被切成两半
    code_chunks = [c for c in chunks if "def critical_logic():" in c]
    assert len(code_chunks) == 1
    assert "print(\"Do not split me!\")" in code_chunks[0]
    assert "```" in code_chunks[0]
```

---

### 3.4 模块四：向量 + BM25 混合检索与重排序（Rerank）评分测试

#### 3.4.1 业务功能与测试目标
混合检索融合了稠密向量检索（语义）与稀疏关键词检索（精确匹配）。测试必须验证：
1. 关键词完全匹配且语义不明显时，BM25 权重保证相关项排前。
2. Rerank 模型计算打分后重新归一化综合得分。

#### 3.4.2 测试用例实现
```python
from core.rag.retrieval import HybridRetriever
from unittest.mock import MagicMock

def test_hybrid_retrieval_and_reranking_score():
    mock_vector_db = MagicMock()
    mock_vector_db.search.return_value = [
        {"id": "doc_1", "score": 0.85, "text": "Python asynchronous programming"},
        {"id": "doc_2", "score": 0.60, "text": "PostgreSQL indexing guide"}
    ]

    mock_bm25 = MagicMock()
    mock_bm25.search.return_value = [
        {"id": "doc_2", "score": 0.95, "text": "PostgreSQL indexing guide"},
        {"id": "doc_3", "score": 0.40, "text": "MySQL database basics"}
    ]

    retriever = HybridRetriever(
        vector_db=mock_vector_db,
        bm25=mock_bm25,
        vector_weight=0.5,
        bm25_weight=0.5
    )

    results = retriever.retrieve(query="PostgreSQL index optimize", top_k=2)

    # 综合两路召回，doc_2 兼备高 BM25 与向量相关，必须名列第一
    assert results[0]["id"] == "doc_2"
    assert results[0]["score"] > results[1]["score"]
```

---

### 3.5 模块五：Workflow DSL YAML 导入导出与模式兼容性测试

#### 3.5.1 业务功能与测试目标
Dify 允许导出应用为 DSL YAML。测试必须保证：
1. 导出 YAML 重新导入后生成的内存图与原图严格等价（Round-trip 一致性）。
2. 缺少必填字段（如节点缺少 ID 或输出端）时，导入引擎精准抛出定位异常。

#### 3.5.2 测试用例实现
```python
import pytest
from core.workflow.dsl import DSLParser, DSLExporter
from core.workflow.errors import InvalidDSLSchemaError

def test_dsl_export_and_import_roundtrip():
    original_dsl_yaml = """
app:
  name: Test AI Workflow
  mode: workflow
workflow:
  version: "0.1.0"
  nodes:
    - id: "start"
      type: "start"
    - id: "end"
      type: "end"
  edges:
    - source: "start"
      target: "end"
"""
    # 1. 导入解析
    workflow = DSLParser.parse(original_dsl_yaml)
    assert len(workflow.nodes) == 2
    assert len(workflow.edges) == 1

    # 2. 导出比对
    exported_yaml = DSLExporter.export(workflow)
    reloaded = DSLParser.parse(exported_yaml)
    assert len(reloaded.nodes) == 2
    assert reloaded.edges[0].source == "start"

def test_dsl_import_schema_validation_error():
    broken_dsl = """
app:
  name: Broken Workflow
workflow:
  nodes:
    - type: "start" # 缺少必填属性 id
"""
    with pytest.raises(InvalidDSLSchemaError) as exc_info:
        DSLParser.parse(broken_dsl)
    assert "Missing required field 'id'" in str(exc_info.value)
```

---

### 3.6 模块六：React Flow 画布交互与端到端运行（E2E）测试

#### 3.6.1 业务功能与测试目标
通过 Playwright 驱动 Web 浏览器测试：
1. 从组件库拖拽节点至画布。
2. 节点端口拉线连接并在非法连接时给出红色阻断提示。
3. 点击“调试运行（Test Run）”抽屉，注入参数并观察节点依次高亮。

#### 3.6.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('Dify Workflow Canvas E2E', () => {
  test('Drag node, connect ports and execute workflow preview', async ({ page }) => {
    await page.goto('http://localhost:3000/app/workflow-test-01/develop');

    // 1. 打开节点侧边栏并拖拽 LLM 节点到画布中间
    const llmNodeItem = page.locator('[data-testid="node-item-llm"]');
    const canvas = page.locator('.react-flow__pane');
    await llmNodeItem.dragTo(canvas, { targetPosition: { x: 400, y: 300 } });

    // 2. 验证画布上出现新挂载的节点
    const newNode = page.locator('.react-flow__node-llm');
    await expect(newNode).toBeVisible();

    // 3. 连线操作：从 Start 节点的输出 Handle 拖到 LLM 节点的输入 Handle
    const startHandle = page.locator('.react-flow__node-start .react-flow__handle-right');
    const llmHandle = newNode.locator('.react-flow__handle-left');
    await startHandle.dragTo(llmHandle);

    // 4. 断言 Edge 连线绘制成功
    await expect(page.locator('.react-flow__edge')).toBeVisible();

    // 5. 点击 Run 抽屉输入变量并运行
    await page.click('[data-testid="workflow-run-btn"]');
    await page.fill('input[placeholder="Enter query"]', 'Hello Dify');
    await page.click('button:has-text("Run Workflow")');

    // 6. 验证执行过程节点状态高亮（Running -> Success）
    await expect(newNode).toHaveClass(/node-running/, { timeout: 5000 });
    await expect(newNode).toHaveClass(/node-success/, { timeout: 15000 });
  });
});
```

---

## 4. 测试工程化与基础设施

1. **Celery 异步队列同步化**：
   * 在 Pytest 环境强制启用 `CELERY_TASK_ALWAYS_EAGER = True` 与 `CELERY_TASK_EAGER_PROPAGATES = True`，消除后台轮询延迟。
2. **轻量内存向量引擎桩**：
   * 采用基于 NumPy 实现的内存余弦相似度类替代外部 Qdrant/Milvus 服务，保证测试不依赖外部复杂集群即可秒级通过。

---

## 5. 对同类复杂系统的参考价值

* **图编排系统的节点状态快照**：测试工作流不能仅看整体最终结果，必须断言图在每个步骤的拓扑状态快照（节点遍历路径、未执行分支的跳过标记），这是保障复杂状态机正确的唯一手段。
* **数据清洗管道的极端语料库回归**：针对自然语言处理系统，维护包含代码、数学公式、极端破损字符的标准测试集，是杜绝版本迭代中分块逻辑退化的核心资产。
