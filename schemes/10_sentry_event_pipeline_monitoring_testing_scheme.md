# Sentry 自动化测试方案与事件监控管道用例全集深度剖析

## 1. 项目背景与系统架构全景

[Sentry](https://github.com/getsentry/sentry) 是全球最顶级的开源应用性能监控（APM）与分布式错误追踪平台。系统采用 Python (Django) + React (TypeScript) 构建主站，底层接入基于 Apache Kafka 的千万级吞吐事件流（Event Stream），并依托 ClickHouse / Snuba 进行超大规模分布式时序分析，配套 Rust 编写的 Symbolicator 高性能二进制符号化服务。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SDK Clients (Browser, Python, Go, iOS)          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ HTTPS POST /api/{project_id}/envelope/
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Relay Proxy (Rust 高吞吐边缘接入网关)                │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Apache Kafka 消息队列
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Sentry Event Processing Pipeline                    │
├───────────────────────────────┬────────────────────────────────────────┤
│ Symbolicator (Rust 服务)      │ Snuba / ClickHouse 分布式时序存储      │
│ - Source Maps 动态解混淆      │ - 百亿级事件秒级过滤与聚合检索         │
│ - iOS dSYM / Android Proguard │ - 慢查询自动分片与缓存优化             │
├───────────────────────────────┼────────────────────────────────────────┤
│ Grouping & Fingerprint Engine │ Alert Rule & Notification Engine       │
│ - 堆栈哈希计算与智能去重合并  │ - 滑动窗口异常率与百分位突增检测       │
│ - 自定义指纹匹配规则          │ - Slack / PagerDuty 报警限频防抖       │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Sentry 拥有在数据管道与高可靠服务中最庞大的全套自动化测试设计：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Relay Ingestion Unit** | Envelope 格式解码、项目 Public Key 鉴权、速率配额限流 | Rust 原生测试 + Python Pytest | 纯内存报文解码，无网络通信 |
| **Symbolication Tests** | Minified JS 结合 Source Map 还原到真实源码行、ELF 符号化 | Python + 独立 Symbolicator 容器 | 本地固定测试 sourcemap 夹具集 |
| **Snuba / ClickHouse Tests** | 复杂事件 SQL 聚合、Tag 过滤、慢查询事务隔离 | Docker Compose (ClickHouse + Snuba) | 测试用例独立 ClickHouse 数据库 |
| **Event Grouping Engine** | 堆栈框架提取、系统函数忽略、指纹计算与 Issue 合并 | Pytest (`tests/sentry/grouping`) | 基于历史真实堆栈的大规模快照回归 |
| **Alert Rules Evaluator** | 滑动窗口错误率计算、动态基线告警、静默期防抖抑制 | Celery 异步队列同步化测试 | 模拟事件发生时间戳注入 |
| **Web UI Dashboard E2E** | 问题详情页堆栈展开、性能火焰图（Flamegraph）查看 | Playwright / Cypress | 隔离的测试组织与虚拟事件注入 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：前端 Source Map 符号化还原（Symbolication）准确性测试

#### 3.1.1 业务功能与测试目标
生产环境的前端代码通常经过 Webpack / Vite 压缩混淆为单行 `app.min.js:1:3452`。Sentry 收到带有压缩堆栈的错误事件时，必须拉取对应的 Source Map 文件，将其还原为源文件 `src/utils/auth.ts:42` 及真实源码上下文。测试必须覆盖：
1. 压缩行列号向原始文件与行号的精确映射。
2. 缺失 Source Map 或 Source Map 破损时的降级处理。

#### 3.1.2 测试用例实现
```python
import pytest
from sentry.models import ReleaseFile
from sentry.lang.javascript.processor import SourceMapProcessor
from sentry.testutils.cases import TestCase

class SourceMapSymbolicationTest(TestCase):
    def test_sourcemap_resolves_minified_stacktrace_to_source_code(self):
        project = self.create_project()
        release = self.create_release(project, version="v1.0.0")

        # 1. 在测试数据库注入模拟的 Source Map 与压缩 JS 静态资源
        minified_content = b"function add(a,b){return a+b}console.log(undefVar);"
        # 对应源码: console.log(undefVar); 位于 src/index.js 第 2 行
        sourcemap_json = b'{"version":3,"file":"min.js","sources":["src/index.js"],"mappings":"AAAA,QAASA,KAAIC,EAAEC,GAAI,MAAOD,GAAEC,EAAhBC,QAAQC,IAARC"}'

        self.create_release_file(project, release, name="~/min.js", file=minified_content)
        self.create_release_file(project, release, name="~/min.js.map", file=sourcemap_json)

        # 2. 模拟前端上报的压缩堆栈帧
        raw_frame = {
            "abs_path": "https://example.com/min.js",
            "filename": "min.js",
            "lineno": 1,
            "colno": 35,
            "function": "add"
        }

        # 3. 执行符号化解析处理器
        processor = SourceMapProcessor(project=project, release=release)
        resolved_frame = processor.process_frame(raw_frame)

        # 4. 严格断言：精确还原文件名、原始代码行号及上下文
        assert resolved_frame["filename"] == "src/index.js"
        assert resolved_frame["lineno"] == 2
        assert "console.log" in resolved_frame.get("context_line", "")
```

---

### 3.2 模块二：事件指纹去重（Grouping & Fingerprinting）聚合算法测试

#### 3.2.1 业务功能与测试目标
成千上万个并发请求抛出同一代码位置的空指针异常时，Sentry 绝不能创建上万个 Issue，必须通过计算堆栈指纹将其聚合进同一个 Issue。同时，若用户配置了自定义指纹（如 `fingerprint: ["my-custom-group"]`），算法必须遵从自定义规则。

#### 3.2.2 测试用例实现
```python
from sentry.grouping.api import get_grouping_calculator
from sentry.testutils.cases import TestCase

class EventGroupingFingerprintTest(TestCase):
    def test_event_stacktrace_grouping_ignores_dynamic_in_app_frames(self):
        calculator = get_grouping_calculator()

        # 构造两个发生时间不同、但根本调用栈相同的事件
        event_a_stack = [
            {"filename": "django/core/handlers/base.py", "function": "get_response", "in_app": False},
            {"filename": "app/views.py", "lineno": 45, "function": "checkout", "in_app": True}
        ]
        event_b_stack = [
            {"filename": "django/core/handlers/base.py", "function": "get_response", "in_app": False},
            {"filename": "app/views.py", "lineno": 45, "function": "checkout", "in_app": True}
        ]

        # 1. 计算两个事件的聚合哈希指纹
        hash_a = calculator.calculate_grouping_hash({"stacktrace": {"frames": event_a_stack}})
        hash_b = calculator.calculate_grouping_hash({"stacktrace": {"frames": event_b_stack}})

        # 2. 强断言：指纹完全相同，必须被聚合
        assert hash_a == hash_b
        assert len(hash_a) == 32 # MD5 长度校验
```

---

### 3.3 模块三：Snuba / ClickHouse 分布式时序查询与慢聚合测试

#### 3.3.1 业务功能与测试目标
验证针对近 24 小时上亿级事件进行 `count() by transaction, project_id` 的查询逻辑，在 ClickHouse 分区表上能够稳定生成合规 SQL，并在超时时间内返回结构化聚合指标。

#### 3.3.2 测试用例实现
```python
from sentry.snuba.dataset import Dataset
from sentry.utils.snuba import raw_snql_query
from sentry.testutils.cases import SnubaTestCase

class SnubaAggregationQueryTest(SnubaTestCase):
    def test_snuba_aggregates_transaction_percentiles(self):
        project = self.create_project()

        # 1. 向受测 ClickHouse 写入测试性能指标事件
        self.store_performance_event(project_id=project.id, transaction="/api/pay", duration=120)
        self.store_performance_event(project_id=project.id, transaction="/api/pay", duration=450)
        self.store_performance_event(project_id=project.id, transaction="/api/pay", duration=80)

        # 2. 构造聚合查询：计算 P95 耗时与总调用量
        query = {
            "dataset": Dataset.Transactions,
            "selected_columns": ["transaction", "p95(duration)", "count()"],
            "groupby": ["transaction"],
            "filter_keys": {"project_id": [project.id]}
        }

        # 3. 执行查询并验证返回数据
        result = raw_snql_query(query)
        data = result["data"]
        assert len(data) == 1
        assert data[0]["transaction"] == "/api/pay"
        assert data[0]["count"] == 3
        assert data[0]["p95_duration"] >= 400
```

---

### 3.4 模块四：Playwright 问题详情页交互与火焰图渲染 E2E 测试

#### 3.4.1 业务功能与测试目标
驱动浏览器访问 Sentry Issue Details 界面：
1. 校验异常概要气泡、标签（Tags）列表展示。
2. 展开完整的调用栈（Stacktrace），验证高亮显示属于本应用（In App）的代码行。
3. 切换到“Tags”选项卡，验证浏览器、操作系统分布饼图渲染。

#### 3.4.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('Sentry Issue Details & Flamegraph E2E', () => {
  test('User views issue details, expands stacktrace and inspects tags', async ({ page }) => {
    await page.goto('http://localhost:8000/organizations/sentry-org/issues/1001/');

    // 1. 验证错误标题与状态展示
    const issueTitle = page.locator('[data-test-id="issue-title"]');
    await expect(issueTitle).toContainText('ZeroDivisionError: division by zero');

    // 2. 点击展开堆栈帧中的原始源码上下文
    const stackFrame = page.locator('.trace-frame:has-text("views.py")').first();
    await expect(stackFrame).toBeVisible();
    await stackFrame.click();

    // 3. 验证源码上下文展开展示对应行号与代码高亮
    const codeContext = stackFrame.locator('.frame-code');
    await expect(codeContext).toBeVisible();
    await expect(codeContext).toContainText('result = 1 / 0');

    // 4. 切换至 Tags 标签页
    await page.click('button:has-text("Tags")');
    await expect(page.locator('[data-test-id="tag-browser"]')).toContainText('Chrome');
  });
});
```

---

## 4. 测试工程化与基础设施

1. **Kafka 内存通道模拟与同步消费**：
   * 在 Pytest 中，通过自定义 Producer/Consumer 内存驱动，拦截所有发往 Kafka 的事件，直接同步灌入 Snuba 存储层，单条端到端用例反馈时间压缩在 50ms 之内。
2. **多租户数据清理隔离**：
   * 使用 `sentry.testutils.cases.TestCase` 自动生成随机 Organization 与 Project ID，并在用例 teardown 阶段批量删除所有关联数据。

---

## 5. 对同类复杂系统的参考价值

* **事件驱动分布式架构的测试闭环**：对于“网关接收 -> 消息队列 -> 异步符号化 -> 分布式存储”的长链路系统，自动化测试必须做到各层独立可测（Mock 下游），同时在集成阶段以轻量受控容器进行全链路连通性穿刺验证。
