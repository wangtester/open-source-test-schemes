# Grafana 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[Grafana](https://github.com/grafana/grafana) 是全球最广泛采用的可观测性与度量分析可视化平台。前端基于 React + TypeScript + Canvas/WebGL 图表引擎，后端采用 Go 编写的高性能 HTTP/WebSocket 数据路由引擎，内置对 Prometheus、Loki、Tempo、PostgreSQL 等数十种异构数据源的统一转换与告警计算。

```
┌────────────────────────────────────────────────────────────────────────┐
│             Grafana Web UI (React + Panel Plugin Engine)               │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ HTTP API / Live WebSocket
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Grafana Core Server (Go)                            │
├───────────────────────────────┬────────────────────────────────────────┤
│ Unified Alerting Engine       │ Dashboard & Folder Permission Engine   │
│ - 告警规则多实例计算调度      │ - RBAC 角色访问控制与权限继承树        │
│ - Alertmanager 路由与静默     │ - 仪表盘 JSON Schema 版本迁移引擎       │
├───────────────────────────────┼────────────────────────────────────────┤
│ Datasource Proxy & Router     │ Live Stream & WebSocket Broker         │
│ - 查询缓存与并发熔断控制     │ - 毫秒级时序指标流式广播推送           │
│ - 异构数据源统一为 Dataframe │ - 分布式频道状态维护                   │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Grafana 的自动化测试体系兼具前端高性能渲染与后端高并发吞吐的全面保障：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Backend Go Unit** | RBAC 权限计算、Dataframe 转换、查询表达式解析、SQL 防注入 | Go `testing` + `testify` | 纯内存计算，无依赖 |
| **Backend DB Integration** | 数据库迁移（SQL 迁移脚本回退）、仪表盘增删改查、事务一致性 | Docker 启动受测 MySQL/Postgres/SQLite | 事务级别回滚或每用例独立 Schema |
| **Alerting Engine Suite** | 持续时序指标计算、Pending 到 Alerting 状态跃迁、路由防抖 | Go 模拟时钟与 Fake Prometheus | 步进式时间推进控制器（Fake Clock） |
| **Frontend Component Tests** | Panel 配置抽屉、变量下拉联动、图表渲染生命周期 | Jest + React Testing Library | Mock 后端 REST API 与 WebSocket |
| **Dashboard Schema Tests** | v5 到 v10+ 历史仪表盘 JSON Schema 的自动迁移与无损升级 | Vitest / Go Schema Validator | 历史各版本 Golden JSON 库回归比对 |
| **Visual & E2E Tests** | 仪表盘网格拖拽吸附、Time Range 框选缩放、Canvas 像素比对 | Playwright + Docker Grafana 集群 | 全局测试实例与隔离测试用户 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：Unified Alerting 告警规则状态跃迁与静默（Silence）引擎测试

#### 3.1.1 业务功能与测试目标
Grafana 告警引擎定期执行查询。当指标超过阈值时，规则进入 `Pending` 状态；如果持续超过 `for: 5m`，状态翻转为 `Alerting` 并发送通知。此外，若命中静默规则（Silence），通知必须被压制。测试必须验证：
1. 状态从 `Normal` -> `Pending` -> `Alerting` 的严格时序跃迁。
2. 静默时间段与标签匹配器（Matchers）的精准阻断。

#### 3.1.2 测试用例实现
```go
package alerting_test

import (
	"context"
	"testing"
	"time"

	"github.com/grafana/grafana/pkg/services/ngalert/eval"
	"github.com/grafana/grafana/pkg/services/ngalert/models"
	"github.com/grafana/grafana/pkg/services/ngalert/state"
	"github.com/stretchr/testify/require"
)

func TestAlertRule_TransitionsFromNormalToPendingToAlerting(t *testing.T) {
	ctx := context.Background()
	fakeClock := time.Now()

	// 1. 构造评估规则：CPU 使用率 > 80%，持续 5 分钟告警
	rule := models.AlertRule{
		ID:           101,
		OrgID:        1,
		Title:        "High CPU Load",
		For:          5 * time.Minute,
		Condition:    "A",
		NoDataState:  models.NoData,
		ExecErrState: models.AlertingErrState,
	}

	stateTracker := state.NewManager()

	// 2. 第一次评估：CPU = 90% (超过阈值)，状态必须跃迁至 Pending
	firstEvalTime := fakeClock
	results := eval.Results{
		eval.Result{Instance: models.AlertLabels{"host": "srv-1"}, State: eval.Alerting, EvaluatedAt: firstEvalTime},
	}
	stateTracker.ProcessEvalResults(ctx, &rule, results, firstEvalTime)

	currState := stateTracker.Get(rule.GetKey(), models.AlertLabels{"host": "srv-1"})
	require.Equal(t, models.AlertStatePending, currState.State)
	require.Equal(t, firstEvalTime, currState.StartsAt)

	// 3. 第二次评估：时间推进 3 分钟 (未达到 5 分钟上限)，状态必须保持 Pending
	stateTracker.ProcessEvalResults(ctx, &rule, results, firstEvalTime.Add(3*time.Minute))
	currState = stateTracker.Get(rule.GetKey(), models.AlertLabels{"host": "srv-1"})
	require.Equal(t, models.AlertStatePending, currState.State)

	// 4. 第三次评估：时间推进 5 分 1 秒，状态必须翻转为 Alerting
	evalAlertingTime := firstEvalTime.Add(5*time.Minute + 1*time.Second)
	stateTracker.ProcessEvalResults(ctx, &rule, results, evalAlertingTime)
	currState = stateTracker.Get(rule.GetKey(), models.AlertLabels{"host": "srv-1"})
	require.Equal(t, models.AlertStateAlerting, currState.State)
}
```

---

### 3.2 模块二：Dashboard 历史版本无损迁移（Migration）回归测试

#### 3.2.1 业务功能与测试目标
Grafana 历经数十个大版本，早期仪表盘的图表类型（如老旧的 Graph Panel、Singlestat）在导入新版系统时，必须被无损转换为现代的 TimeSeries、Stat 或 BarChart Panel，且数据源绑定、阈值颜色配置绝对不能丢失。

#### 3.2.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';
import { migrateDashboardJson } from 'app/features/dashboard/state/DashboardMigration';
import v6SinglestatDashboard from './fixtures/dashboard_v6_singlestat.json';

describe('Dashboard Migration Engine Suite', () => {
  it('migrates legacy v6 singlestat panel into modern stat panel without data loss', () => {
    // 1. 加载包含历史 singlestat 的原始仪表盘 JSON
    const legacyDashboard = JSON.parse(JSON.stringify(v6SinglestatDashboard));
    expect(legacyDashboard.panels[0].type).toBe('singlestat');
    expect(legacyDashboard.schemaVersion).toBe(16);

    // 2. 执行自动升级迁移函数
    const migrated = migrateDashboardJson(legacyDashboard);

    // 3. 结构化严格断言
    expect(migrated.schemaVersion).toBeGreaterThanOrEqual(30);
    const modernPanel = migrated.panels[0];
    
    // 面板类型升级为 modern stat
    expect(modernPanel.type).toBe('stat');
    // 颜色阈值区间自动映射为 standard.thresholds
    expect(modernPanel.fieldConfig.defaults.thresholds.steps).toEqual([
      { value: null, color: 'green' },
      { value: 80, color: 'red' },
    ]);
    // 维持历史数据源引用
    expect(modernPanel.datasource).toEqual(legacyDashboard.panels[0].datasource);
  });
});
```

---

### 3.3 模块三：数据源代理（Datasource Proxy）高并发熔断与查询防穿透测试

#### 3.3.1 业务功能与测试目标
当用户在大型看板刷新数据时，可能同一秒触发几十个 Panel 的同时查询。Grafana 后端代理必须具备单数据源最大并发限制与查询排队超时能力，防止把上游 Prometheus/MySQL 打垮。

#### 3.3.2 测试用例实现
```go
package proxy_test

import (
	"context"
	"net/http"
	"net/http/httptest"
	"sync"
	"sync/atomic"
	"testing"
	"time"

	"github.com/grafana/grafana/pkg/services/datasources"
	"github.com/stretchr/testify/require"
)

func TestDatasourceProxy_ConcurrencyLimitAndQueueing(t *testing.T) {
	var activeUpstreamRequests int32
	var maxObservedConcurrency int32

	// 1. 模拟上游较慢的数据源服务器
	mockServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		current := atomic.AddInt32(&activeUpstreamRequests, 1)
		for {
			max := atomic.LoadInt32(&maxObservedConcurrency)
			if current <= max || atomic.CompareAndSwapInt32(&maxObservedConcurrency, max, current) {
				break
			}
		}
		time.Sleep(50 * time.Millisecond)
		atomic.AddInt32(&activeUpstreamRequests, -1)
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`{"status":"success"}`))
	}))
	defer mockServer.Close()

	// 2. 配置数据源：限制最大并发连接数为 2
	ds := &datasources.DataSource{
		ID:        1,
		URL:       mockServer.URL,
		JsonData:  map[string]any{"maxConnsPerHost": 2},
	}

	proxy := datasources.NewProxy(ds)
	var wg sync.WaitGroup
	var errorCount int32

	// 3. 并发派发 10 个查询请求
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			req := httptest.NewRequest("GET", "/api/datasources/proxy/1/query", nil)
			w := httptest.NewRecorder()
			proxy.HandleRequest(context.Background(), w, req)
			if w.Code != http.StatusOK {
				atomic.AddInt32(&errorCount, 1)
			}
		}()
	}
	wg.Wait()

	// 4. 断言：所有请求最终均成功，但同时打入上游的最大并发数严格 <= 2
	require.Equal(t, int32(0), errorCount)
	require.LessOrEqual(t, maxObservedConcurrency, int32(2))
}
```

---

### 3.4 模块四：Playwright 仪表盘网格拖拽（React-Grid-Layout）与时间框选 E2E 测试

#### 3.4.1 业务功能与测试目标
测试核心交互场景：
1. 在仪表盘上拖拽调整 Panel 大小，网格能自动重新吸附计算并保存坐标（`x, y, w, h`）。
2. 在 TimeSeries Panel 图表 Canvas 上按下鼠标框选特定时间区间，全局 TimeRange Picker 自动缩放对应时间。

#### 3.4.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('Grafana Dashboard Layout and TimeRange E2E', () => {
  test('User resizes panel and zooms into time range on chart', async ({ page }) => {
    await page.goto('http://localhost:3000/d/prod-metrics/production-overview');

    // 1. 等待仪表盘与图表渲染完成
    const chartPanel = page.locator('[data-testid="panel-container-1"]');
    await expect(chartPanel).toBeVisible();

    // 2. 拖拽调整 Panel 尺寸：抓取右下角 Resize Handle 并拖拽
    const resizeHandle = chartPanel.locator('.react-resizable-handle');
    const initialBox = await chartPanel.boundingBox();
    
    await resizeHandle.hover();
    await page.mouse.down();
    await page.mouse.move(initialBox!.x + initialBox!.width + 100, initialBox!.y + initialBox!.height + 50);
    await page.mouse.up();

    // 验证面板新尺寸生效
    const updatedBox = await chartPanel.boundingBox();
    expect(updatedBox!.width).toBeGreaterThan(initialBox!.width);

    // 3. 在 Canvas 图表上框选放大时间范围
    const canvas = chartPanel.locator('canvas.u-over');
    const canvasBox = await canvas.boundingBox();
    
    // 鼠标在图表 20% ~ 80% 横坐标范围内划选
    await page.mouse.move(canvasBox!.x + canvasBox!.width * 0.2, canvasBox!.y + canvasBox!.height * 0.5);
    await page.mouse.down();
    await page.mouse.move(canvasBox!.x + canvasBox!.width * 0.8, canvasBox!.y + canvasBox!.height * 0.5);
    await page.mouse.up();

    // 4. 断言顶部全局时间选择器更新为精确的自定义时间区间
    const timePickerBtn = page.locator('[data-testid="time-picker-button"]');
    await expect(timePickerBtn).not.toContainText('Last 6 hours');
  });
});
```

---

## 4. 测试工程化与基础设施

1. **虚拟时钟推进（Fake Clock Driven Scheduling）**：
   * 告警引擎测试绝对禁止使用 `time.Sleep`，全线采用基于虚拟时钟接口注入，直接调用 `clock.Advance(5 * time.Minute)` 瞬间完成状态验证。
2. **多数据库自动化矩阵回归**：
   * 数据库迁移测试在 CI 中通过 Docker 矩阵同时拉起 MySQL 8.0、PostgreSQL 15 以及 SQLite，确保同一套 SQL DDL 脚本跨方言无歧义执行。

---

## 5. 对同类复杂系统的参考价值

* **时间密集型状态机的测试绝招**：任何包含“持续 N 分钟报警/熔断/降级”的业务系统，必须把时间源（Clock）抽象为可注入接口，测试用例通过直接快进虚拟时间来进行确定性断言。
* **数据流聚合与管道代理隔离**：为外部慢速资源建立代理并发测试，模拟高并发下的限流与排队，是保障高可用中间件的关键手段。
