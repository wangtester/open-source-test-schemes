# FastGPT 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[FastGPT](https://github.com/labring/FastGPT) 是开源界标杆级的 AI 知识库问答与复杂对话流程编排系统。系统前后端均采用 TypeScript (Next.js) 开发，架构设计与实现中广泛使用 AI 智能体辅助生成。底层数据中枢依赖 MongoDB（关系元数据与复杂聚合流）与向量数据库（Milvus / PGVector）。

### 1.2 被测系统核心组成
1. **多格式文件分块解析管道（File Ingestion Pipeline）**：PDF, DOCX, CSV, Markdown 的多段落切片、自动清洗与去重（Hash Deduplication）。
2. **混合检索与重排序引擎（Hybrid Search & Rerank）**：向量相似度匹配 + BM25 全文检索加权融合，并调用 Rerank 模型二次重排。
3. **流程节点编排调度（Visual Workflow Dispatcher）**：基于 React Flow 的模块化节点（知识库搜索、多轮对话、问题分类、HTTP 插件）。
4. **SSE 流式传输与 Token 计量（Streaming Output & Token Billing）**：实时打字机分块流下发与用户余额/配额原子扣减。

### 1.3 自动化测试的主要难点与挑战
* **MongoDB 复杂聚合管道（Aggregation Pipeline）测试**：知识库召回涉及大量 `$match`, `$lookup`, `$facet` 操作，需验证多表聚合结果正确性。
* **向量检索评测指标量化**：如何确定性评估召回命中率（Hit Rate）与排序位置（MRR）。
* **流式传输中途断开时的事务回滚**：客户端关闭页面后，已消耗 Token 必须精准记录，后续任务必须立刻停止。

---

## 2. 测试架构全景与分层策略

FastGPT 建立了以 MongoDB 内存副本集为基础的流式自动化测试体系：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Workflow & Chat E2E             │  前端全流程：驱动知识库上传、流式会话与流程画布
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         SSE Streaming & Token Accounting Tests         │  协议与计费层：捕获 EventSource 流分块并断言积分原子扣减
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│     Hybrid Search & Vector Recall Integration Suite    │  检索计算层：断言 BM25 + 向量检索混合权重打分算法
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│        MongoMemoryServer & Ingestion Unit Tests        │  数据底座：基于内存版 MongoDB 的纯净集合隔离测试
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：混合检索（Hybrid Search）权重融合算法测试

#### 3.1.1 业务功能背景
用户提问时，系统分别发起向量搜索（获取前 20 条）与关键词搜索（获取前 20 条），通过 RRF（Reciprocal Rank Fusion）或加权公式融合为前 5 条送入大模型。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Vitest 单元测试）**：
```typescript
import { describe, it, expect } from 'vitest';
import { calculateRRFScore } from '@/service/core/dataset/search';

describe('Hybrid Search Fusion Algorithm', () => {
  it('correctly ranks items appearing in both vector and keyword results at top', () => {
    const vectorResults = [{ id: 'doc-1', score: 0.95 }, { id: 'doc-2', score: 0.88 }];
    const keywordResults = [{ id: 'doc-2', score: 10.5 }, { id: 'doc-3', score: 8.2 }];

    // 计算融合排序结果
    const fused = calculateRRFScore({ vectorResults, keywordResults, k: 60 });

    // 断言：同时被两路检索命中的 doc-2 综合排名第一
    expect(fused[0].id).toBe('doc-2');
    expect(fused[0].finalScore).toBeGreaterThan(fused[1].finalScore);
  });
});
```

---

### 3.2 功能二：SSE 流式响应断开与计费原子性断言

#### 3.2.1 业务功能背景
当用户发起会话，模型流式吐出第 5 个 Chunk 时，用户关闭浏览器。系统必须停止向外部大模型发请求，并原子更新用户账单表，按实际已吐出的 5 个 Chunk 扣减余额。

#### 3.2.2 测试用例设计方案
* **测试编排**：
  1. 通过测试 HTTP 客户端发起 POST 请求并获取 ReadableStream。
  2. 读取前 2 个 Chunk。
  3. 客户端主动执行 `stream.cancel()`。
  4. 检查用户数据库记录中的 `user.points` 余额。
  5. **断言**：余额按前 2 个 Chunk 精准扣减，没有多扣也没有发生负数穿透。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **MongoMemoryServer 快速环境拉起**：
   * 所有测试使用 `mongodb-memory-server` 在内存中虚拟出完整的 Mongo 实例，每个测试套件结束后执行 `db.dropDatabase()`，无任何磁盘清理负担。

---

## 5. 对复杂系统的实战借鉴意义

* **计费与计量系统必须结合“流中断测试”**：AI 时代按 Token 计费是核心商业闭环，必须在测试中显式模拟断网和主动取消，验证用量计算的绝对精确。
* **算法层优先进行纯数学测试**：对于混合打分、分块分词算法，将其抽离为无外部数据库依赖的纯函数，能让测试执行速度提升百倍。
