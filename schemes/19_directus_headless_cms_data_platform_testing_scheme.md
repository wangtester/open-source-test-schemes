# Directus 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Directus](https://github.com/directus/directus) 是全球领先的开源实时数据平台与无代码 Headless CMS。系统后端采用 Node.js (TypeScript) 构建于 Knex.js 之上，前端采用 Vue 3 + Pinia 构建现代化 SPA 工作台，能够跨接并适配所有主流 SQL 关系数据库（PostgreSQL, MySQL, SQLite, Oracle, Microsoft SQL Server, CockroachDB）。

### 1.2 被测系统核心组成
1. **多视图自适应工作台（Multi-layout Data Studio）**：允许用户在表格视图（Table）、看板视图（Kanban）、日历视图（Calendar）、卡片视图（Cards）和地图视图（Map）之间一键切换并交互式编辑数据。
2. **字段级细粒度权限（Field-Level Security & Conditional Rules）**：精确控制不同角色对某个字段的读、写、校验与动态字段掩码（Data Masking）。
3. **数据模型构建器（Visual Schema Builder）**：在 Web 界面创建集合（Collection）、增删字段、配置 M2M/O2M/M2O 复杂关联关系并自动执行底层 DDL 变更。
4. **实时事件与自动化工作流（Flows & Webhooks）**：可视化低代码触发器与动作编排引擎。

### 1.3 自动化测试的主要难点与挑战
* **跨 6 种数据库物理特性的兼容性鸿沟**：同一种数据类型（如 JSON、UUID、Auto-increment、Geometry）在 SQLite 与 Oracle/SQL Server 上的实现差异极大。
* **复杂 Vue 3 前端多视图交互的组件生命周期**：看板视图的拖拽跨列（Drag-to-status）、表格视图的列宽拖拽与行级勾选。
* **字段级权限与条件规则的排列组合爆炸**：一个字段在“状态为草稿时可写，状态为已发布时只读且对财务人员隐藏”的矩阵测试。

---

## 2. 测试架构全景与分层策略

Directus 演进出了一套兼顾多数据库矩阵与前端复杂交互的测试架构体系：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Vue 3 Studio E2E Suite          │  前端交互层：驱动看板拖拽、多视图切换、动态表单渲染
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Vitest Multi-Vendor Database Matrix Tests         │  多库矩阵层：同一套测试用例并行在 6 种真实数据库容器运行
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│         Schema DDL & Knex Query Abstraction Tests      │  模式迁移层：验证动态字段创建、重命名与外键关系
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Field-Level Permissions & Auth Logic           │  权限控制层：组合用户角色与字段遮掩规则进行契约断言
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：看板视图（Kanban Layout）交互拖拽与状态机更新

#### 3.1.1 业务功能背景
用户进入“任务管理”集合，切换为看板视图，将一张卡片从“进行中（In Progress）”列拖拽释放到“已完成（Done）”列。前端应触发卡片位置重绘，并向后端发送 PATCH 请求将该任务的 `status` 字段修改为 `done`。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Playwright E2E 交互测试）**：
```typescript
test('drag card in kanban view updates item status', async ({ page }) => {
  await page.goto('/admin/content/tasks');

  // 1. 切换布局为 Kanban 看板
  await page.click('button[aria-label="Layout Options"]');
  await page.click('button:has-text("Kanban")');

  // 2. 定位受测卡片与目标分组列
  const card = page.locator('.kanban-card:has-text("Implement OAuth")').first();
  const doneColumn = page.locator('.kanban-column[data-status="done"]');

  // 3. 执行拖拽并放置（Drag and Drop）
  await card.dragTo(doneColumn);

  // 4. 拦截并断言向后端发送的 PATCH 更新请求
  const patchRequest = await page.waitForRequest(req => 
    req.url().includes('/items/tasks/') && req.method() === 'PATCH'
  );
  expect(patchRequest.postDataJSON()).toEqual(
    expect.objectContaining({ status: 'done' })
  );

  // 5. 验证 UI 界面上该卡片已被挂载在 Done 列 DOM 树下
  await expect(doneColumn.locator('.kanban-card:has-text("Implement OAuth")')).toBeVisible();
});
```

---

### 3.2 功能二：字段级权限（Field-Level Security）与数据掩码测试

#### 3.2.1 业务功能背景
针对普通职员角色（`Employee`），配置对员工表（`employees`）的权限为：允许读取 `name` 和 `department`，但强制隐藏 `salary`（字段级不可读），且对 `ssn`（社会保障号）应用掩码规则只保留后四位。

#### 3.2.2 测试用例设计方案
* **测试用例实现（Vitest 后端集成测试）**：
```typescript
import { describe, it, expect } from 'vitest';

describe('Field-Level Security (FLS) & Data Masking', () => {
  it('strips restricted fields and applies masking rule based on user role', async () => {
    // 1. 模拟以 Employee 角色登录
    const employeeClient = await createTestClient({ role: 'employee' });

    // 2. 发起查询所有员工数据的接口请求
    const response = await employeeClient.get('/items/employees?fields=*');
    expect(response.status).toBe(200);

    const firstItem = response.data.data[0];

    // 3. 断言允许的字段正常返回
    expect(firstItem.name).toBe('Alice Smith');
    expect(firstItem.department).toBe('Engineering');

    // 4. 硬性断言：受限制的敏感字段必须在 JSON 中彻底不存在（被剔除），而非返回 null
    expect(firstItem).not.toHaveProperty('salary');

    // 5. 断言掩码转换生效：前缀脱敏
    expect(firstItem.ssn).toBe('***-**-6789');
  });
});
```

---

### 3.3 功能三：动态数据模型 DDL 热变更兼容性验证

#### 3.3.1 业务功能背景
管理员在 UI 上点击“创建字段”，添加一个名为 `rating` 的浮点数字段，配置默认值为 `5.0`。Directus 内部由 Schema Inspector 检查元数据，并由 Knex 向底层物理表下发 `ALTER TABLE ... ADD COLUMN ...`。

#### 3.3.2 测试用例设计方案
* **测试编排**：
  1. 调用 API 动态为已有集合创建新字段。
  2. 绕过 Directus 应用层，直接通过原生数据库驱动连接执行 `SELECT rating FROM ...`。
  3. 写入一条缺失该字段的新记录，断言读取时数据库底层默认值 `5.0` 自动生效。
  4. 随后调用删除字段 API，断言物理列被干净删除，且外键约束无残留。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **跨 6 种主流 SQL 数据库容器并行矩阵**：
   * 在 CI 中启动 PostgreSQL 15, MySQL 8, MariaDB 10, MSSQL 2022, Oracle XE, SQLite 内存模式。测试套件同一套逻辑在所有容器上执行，对底层 SQL 方言生成做 100% 交叉比对。
2. **Schema 隔离与自动化回滚**：
   * 采用前缀隔离（如 `directus_test_run_${uuid}_`），防止并发测试运行器发生表名冲突。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **无代码/低代码平台必须设计“数据抽象层驱动矩阵测试”**：对于支持多种数据库或外部存储的平台，单一数据库测试完全不够，必须通过容器矩阵验证数据类型映射与 DDL 的物理执行。
* **数据安全测试应坚持“键不存在（Non-existent Key）”原则**：测试字段级权限时，不仅要验证空值，更要验证键名未在返回的 JSON 树中序列化，防止未授权字段存在性泄露。
