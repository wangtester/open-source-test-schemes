# Grafana 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Grafana](https://github.com/grafana/grafana) 是全球最普及的操作可视化与指标监控大盘平台。后端采用 Go 语言构建，前端采用 React、TypeScript、Redux Toolkit 与基于 WebGL/Canvas 的高性能图表引擎，支持对接 Prometheus、Loki、Elasticsearch、InfluxDB、PostgreSQL 等数十种异构数据源。

### 1.2 被测系统核心组成
1. **仪表盘编排网格（Dashboard Grid & Panel Layout）**：支持在二维网格内任意拖拽、拉伸缩放、复制面板、调整图表类型。
2. **多维模板变量与级联下钻（Template Variables & Query Interpolation）**：支持定义动态变量（如 `$region`, `$cluster`, `$pod`），下拉联动触发全页面图表自适应刷新。
3. **统一告警评估引擎（Unified Alerting Engine）**：后台 Go 服务按周期提取时序数据点，执行表达式计算、滑动窗口统计与状态机翻转。
4. **数据源代理与转换管道（Data Source Proxy & Transformations）**：提供多源聚合计算（如将 Prometheus 与 MySQL 数据按时间戳 Join）。

### 1.3 自动化测试的主要难点与挑战
* **极度复杂的可视化画布交互（Drag, Drop & Resize）**：面板在网格中的拖拽受 CSS Grid、绝对定位坐标与节流渲染影响，极易在自动化操作中失真。
* **高频时序数据动态刷新与图表渲染同步**：图表采用 Canvas/SVG 混合渲染，普通的 DOM 文本匹配无法直接验证折线图与柱状图的绘制准确性。
* **多层级异步数据流下推**：一次变量切换可能触发几十个后端数据源并发请求，需验证并发请求无竞争、无内存泄漏。

---

## 2. 测试架构全景与分层策略

Grafana 团队打造了一套覆盖全栈的测试体系：

```
┌────────────────────────────────────────────────────────┐
│             Playwright / Cypress E2E Suite             │  前端全景层：真实交互模拟拖拽网格、时间范围选取、导出
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Panel & Component RTL Unit Tests (Jest)           │  前端组件层：React Testing Library 驱动局部面板与表单
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Go Backend Integration Tests (Database/HTTP)   │  后端集成层：直接驱动 Go HTTP 路由与数据源插件管道
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│        Alerting & Expression Go Unit Engine            │  引擎计算层：纯算法验证滑动窗口与数学表达式计算
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：仪表盘网格拖拽重排与坐标持久化（Grid Drag-and-Drop）

#### 3.1.1 业务功能背景
用户长按面板头部拖拽至新网格位置，或者拖拽右下角拉大面板尺寸。系统必须实时重新计算网格碰撞碰撞检测算法，并在用户点击保存时将变更后的坐标序列化为 Dashboard JSON 提交至后端。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Playwright E2E 交互测试）**：
```typescript
test('should drag panel to new position and persist updated grid coordinates', async ({ page }) => {
  await page.goto('/d/my-test-dashboard');

  // 1. 定位受测面板与拖拽把手
  const panelHeader = page.locator('[data-panelid="2"] .panel-title-container');
  const targetGridCell = page.locator('.react-grid-layout');

  // 2. 模拟高精度鼠标轨迹拖拽（Mouse Drag Actions）
  const box = await panelHeader.boundingBox();
  expect(box).not.toBeNull();

  await page.mouse.move(box!.x + box!.width / 2, box!.y + box!.height / 2);
  await page.mouse.down();
  // 沿 Y 轴向下拖动 300 像素至下一行网格槽位
  await page.mouse.move(box!.x + box!.width / 2, box!.y + 300, { steps: 10 });
  await page.mouse.up();

  // 3. 触发保存大盘弹窗
  await page.click('button[aria-label="Save dashboard"]');
  await page.click('button[data-testid="confirm-save-dashboard-button"]');

  // 4. 验证网络持久化 Payload：拦截并断言提交给后端的 JSON 坐标
  const saveRequest = await page.waitForRequest(req => req.url().includes('/api/dashboards/db') && req.method() === 'POST');
  const postData = saveRequest.postDataJSON();
  const movedPanel = postData.dashboard.panels.find((p: any) => p.id === 2);
  
  // 断言网格 Y 轴坐标从原来的 0 改变为下方的值
  expect(movedPanel.gridPos.y).toBeGreaterThan(0);
});
```

---

### 3.2 功能二：模板变量级联下钻与请求参数自动插值

#### 3.2.1 业务功能背景
大盘顶部有 `$datacenter` 和 `$server` 两个下拉框。当用户选择 `$datacenter = "us-east-1"` 时，`$server` 下拉框必须自动发起查询请求过滤候选列表，同时下方图表的 Prometheus 查询由 `cpu_util{dc="$datacenter"}` 实时插值为 `cpu_util{dc="us-east-1"}`。

#### 3.2.2 测试用例设计方案
* **测试用例实现（React Testing Library + Mock Service Worker）**：
```typescript
test('selecting variable updates downstream dependent variable and triggers query', async () => {
  // 1. 渲染包含变量控制条的大盘页面组件
  render(<DashboardPage dashboardId="test-vars" />);

  // 2. 模拟点击 $datacenter 变量下拉菜单并选择选项
  const dcSelect = screen.getByRole('combobox', { name: 'Variable DataCenter' });
  fireEvent.change(dcSelect, { target: { value: 'us-east-1' } });

  // 3. 断言下级变量 $server 的选项正在异步刷新
  await waitFor(() => {
    const serverOptions = screen.getAllByRole('option');
    expect(serverOptions.map(o => o.textContent)).toEqual(['server-us-1', 'server-us-2']);
  });

  // 4. 检查后端查询客户端捕获的实际网络参数
  expect(mockDatasourceQuery).toHaveBeenCalledWith(
    expect.objectContaining({
      targets: [
        expect.objectContaining({
          expr: 'cpu_util{dc="us-east-1"}', // 验证已完成变量替换
        }),
      ],
    })
  );
});
```

---

### 3.3 功能三：后端统一告警引擎滑动窗口状态机测试

#### 3.3.1 业务功能背景
告警规则配置为：`avg() OVER 5m > 80%`，每 10 秒执行一次评估。连续 3 次评估超标进入 `Pending`，持续 1 分钟超标进入 `Firing` 并触发 Webhook。

#### 3.3.2 测试用例设计方案
* **测试用例实现（Go 单元与时序计算测试）**：
```go
func TestAlertRuleEvaluationStateTransitions(t *testing.T) {
    evaluator := alerting.NewRuleEvaluator()
    rule := alerting.Rule{
        Condition: "avg() > 80",
        ForDuration: 1 * time.Minute,
    }

    // 第一次评估：值 85% -> 翻转为 Pending
    state := evaluator.Eval(rule, 85.0, time.Now())
    assert.Equal(t, alerting.StatePending, state.Current)

    // 第二次评估：时间未到 1 分钟 -> 维持 Pending
    state = evaluator.Eval(rule, 88.0, time.Now().Add(30*time.Second))
    assert.Equal(t, alerting.StatePending, state.Current)

    // 第三次评估：时间超过 1 分钟且持续超标 -> 翻转为 Firing
    state = evaluator.Eval(rule, 92.0, time.Now().Add(65*time.Second))
    assert.Equal(t, alerting.StateFiring, state.Current)
    assert.True(t, state.ShouldNotify) // 断言生成发送通知指令
}
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Go Race Detector 常态化启用**：
   * 后端所有自动化测试默认开启 `go test -race`，在每次提交中自动探测数据源并发查询与告警调度中的 Goroutine 内存数据竞态。
2. **多数据库后端驱动矩阵（SQLite, MySQL 8.0, PostgreSQL 15）**：
   * 针对元数据存储层，CI 矩阵同时在三套数据库实例上运行全量集成测试，确保大盘 JSON 字段类型与自增主键一致性。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **复杂拖拽网格的自动化必须结合“坐标拦截校验”**：单纯断言 UI 视觉可能存在微小偏差，在操作触发后断言提交给后端的实体坐标/网格排版结构体是杜绝排版 Bug 的最稳妥方法。
* **模板/变量引擎的核心是“插值表达式纯函数测试”**：将 UI 交互事件与底层的模板字符串求值引擎解耦，先把变量替换与级联逻辑做成 100% 覆盖的纯计算单元测试，再做端到端集成。
