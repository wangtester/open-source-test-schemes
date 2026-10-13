# Khoj 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Khoj](https://github.com/khoj-ai/khoj) 是开源领域的下一代个人 AI 知识助手与长程记忆（Long-term Memory）系统（由 Y Combinator 孵化）。其项目代码与自动化流水线由核心团队依托 AI 智能体协同研发。系统后端基于 Python (FastAPI) + Pydantic + SQLAlchemy，前端基于 React，能够实时或定时同步用户在 Notion、GitHub、Obsidian、PDF 与本地文件夹中的知识碎片，构建个性化动态记忆库。

### 1.2 被测系统核心组成
1. **多数据源增量同步管道（Incremental Sync Pipeline）**：周期性抓取多平台异构文件，利用内容哈希指纹（Content Hash）检测新增、修改与删除。
2. **混合长程记忆检索（Semantic & Recency Memory Retrieval）**：结合语义相似度与时间衰减函数（Time Decay Factor）动态计算记忆唤醒得分。
3. **自动化任务调度与代理执行（Automations & Agent Schedules）**：后台 Cron 周期任务调度、网页爬虫与邮件/消息定时推送。
4. **Web 知识管理控制台**：多源连接器配置、会话式知识探寻与记忆碎片审查。

### 1.3 自动化测试的主要难点与挑战
* **大量外部第三方平台（Notion, GitHub, 邮箱）的 API 依赖**：离线持续集成中如何逼真模拟多数据源授权与文档内容拉取。
* **增量同步状态机一致性断言**：当用户在本地删除或重命名文件时，底层向量数据库与关系型元数据表中对应的记忆条目能否干净同步抹除。
* **记忆检索相关性评测（Recall Benchmarking）的非确定性**：如何避免算法调整导致的历史知识召回率退化。

---

## 2. 测试架构全景与分层策略

Khoj 建立了以离线数据管道与评测基准为核心的测试架构：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Knowledge Dashboard E2E         │  前端交互层：驱动数据源绑定、交互式记忆探索与设置
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Incremental Sync Pipeline Integration Tests       │  管道集成层：模拟文件创建/修改/删除，断言索引增量更新
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Memory Scoring & Time Decay Unit Tests         │  算法单元层：纯数学断言记忆相关性与时间衰减权重组合
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│     In-Memory SQLite & Chroma Vector Testbed           │  数据底座：轻量级内存数据库，测试用例执行完全无物理 IO
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：多源文件增量同步与删除清理断言

#### 3.1.1 业务功能背景
用户配置了本地笔记目录。首次同步拉取 10 份文件并生成向量索引；随后用户修改了 `notes.md` 并删除了 `draft.md`。执行同步时，系统必须仅更新 `notes.md` 的嵌入向量，同时必须将 `draft.md` 从关系型数据库和向量表中物理清除。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Pytest 数据管道集成测试）**：
```python
import pytest
from khoj.processor.content.text.markdown_processor import MarkdownProcessor
from khoj.database.models import Entry

def test_incremental_sync_updates_and_deletes_entries(test_db_session):
    processor = MarkdownProcessor(session=test_db_session)

    # 1. 第一轮同步：载入 2 个文件
    initial_files = {
        "/docs/note.md": "Alpha content about Python",
        "/docs/draft.md": "Temporary notes"
    }
    processor.sync(initial_files)
    assert test_db_session.query(Entry).count() == 2

    # 2. 第二轮增量变更：修改 note.md，删除 draft.md
    updated_files = {
        "/docs/note.md": "Alpha content updated with Advanced Python"
    }
    processor.sync(updated_files)

    # 3. 严格断言：总条目变为 1，且 note.md 内容已物理更新
    assert test_db_session.query(Entry).count() == 1
    remaining_entry = test_db_session.query(Entry).first()
    assert remaining_entry.file_path == "/docs/note.md"
    assert "Advanced Python" in remaining_entry.raw_text
```

---

### 3.2 功能二：记忆唤醒得分（时间衰减 + 语义相似度）计算测试

#### 3.2.1 业务功能背景
两条笔记内容相近（语义相似度均为 0.85），但笔记 A 记录于 2 小时前，笔记 B 记录于 3 年前。在记忆召回排序中，笔记 A 的最终唤醒得分必须高于笔记 B。

#### 3.2.2 测试用例设计方案
* **测试用例实现**：
```python
from datetime import datetime, timedelta
from khoj.search_filter.memory import score_memory_relevance

def test_memory_recency_decay_boost():
    now = datetime.utcnow()
    recent_time = now - timedelta(hours=2)
    old_time = now - timedelta(days=1000)

    score_recent = score_memory_relevance(sim_score=0.85, created_at=recent_time, current_time=now)
    score_old = score_memory_relevance(sim_score=0.85, created_at=old_time, current_time=now)

    # 断言最近的记忆获得时间增益
    assert score_recent > score_old
    assert score_recent - score_old > 0.1
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **零外部网络隔离执行（Hermetic Tests）**：
   * 所有第三方外部数据源（GitHub API, Notion API）均采用自研的响应假数据生成器，断绝 CI 与公网连通，提升执行速度与抗抖动能力。
2. **轻量向量检索内存模式**：
   * 测试套件使用 ChromaDB 的 EphemeralClient（纯内存模式），用完即弃，避免本地遗留缓存文件。

---

## 5. 对复杂系统的实战借鉴意义

* **增量数据同步必须测试“删除（Tombstone / Deletion）”闭环**：很多系统往往只测新增和修改，导致大量僵尸孤儿数据留在向量库中；在增量测试中验证已删除文件的及时清退是核心质量防线。
* **将多因子打分算法抽象为确定性纯函数**：涉及时间衰减、权重加权、综合排名的复杂系统，测试应通过显式传入基准时间点（`current_time`）进行确定性比对，杜绝因当前系统时间漂移引发的断言波动。
