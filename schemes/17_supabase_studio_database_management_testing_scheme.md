# Supabase Studio 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Supabase](https://github.com/supabase/supabase) 是全球顶级的开源 Firebase 替代品。其核心控制台 **Supabase Studio** 是一个极为强大的 Web 数据库工作台，基于 Next.js、React、Tailwind CSS 与 Monaco Editor 构建。底层深度依托 PostgreSQL、PostgREST（RESTful 适配层）、GoTrue（身份认证）与 Realtime（WAL 逻辑复制流广播）。

### 1.2 被测系统核心组成
1. **Web 表格编辑器（Table Editor & Grid View）**：如同 Excel 般的强大网格界面，支持零代码添加表/列、修改数据类型、编辑外键约束、行级数据双击即改。
2. **可视化 RLS 策略构建器（Row Level Security Policy Builder）**：在网页端以可视化表单配置复杂的 Postgres 行级安全策略，并自动生成校验 SQL。
3. **交互式 SQL 编辑器（SQL Editor & Snippets）**：支持多标签页、代码补全、执行结果表格化与执行耗时分析。
4. **实时复制与数据库配置（Replication & Extension Manager）**：在界面一键启用 pgvector 向量插件、PostGIS 空间地理插件与 WAL 日志广播。

### 1.3 自动化测试的主要难点与挑战
* **前端网格虚拟化滚动（Virtual Scroll）与输入失焦**：包含数十万行数据的表格只在可视区域渲染数十个 DOM 节点，自动化脚本在滚动、编辑失焦（Blur）时极易丢失状态。
* **对底层 PostgreSQL 真实元数据（Catalog）的实时同步**：前端修改了一列的名称，后端不仅是更新业务表，而是真实执行了 `ALTER TABLE ...` DDL 语句，需验证 DDL 执行成功且不锁死数据库。
* **RLS 策略生效与多租户权限断言的闭环**：在界面配置完 RLS 后，需要立即用两个不同角色的 JWT Token 发起数据请求验证隔离效果。

---

## 2. 测试架构全景与分层策略

Supabase 打造了“前端控制台交互 + 数据库内核 pgTAP”的双重校验闭环：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Studio Web E2E Suite            │  网页交互层：驱动虚拟滚动表格编辑、RLS 表单配置、SQL 执行
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│         PostgREST API & OpenAPI Schema Tests           │  接口转换层：验证 DDL 变更后 REST API Schema 实时同步
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      pgTAP In-Database SQL Assertion Suite             │  数据库内核层：直接在 Postgres 内部运行 SQL 断言函数
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Supabase Local CLI Docker Environment          │  环境底座：通过 supabase start 毫秒级拉起完整本地栈
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：Web 表格编辑器交互式单元格就地编辑（Inline Cell Editing）

#### 3.1.1 业务功能背景
用户在 Studio 的表格界面双击某一行的 `user_email` 单元格，输入新的邮箱地址并按下回车或点击空白处失焦。前端应立即发起 PATCH 请求更新数据库，并在成功后更新单元格高亮状态，若格式不合法（如违反唯一约束或 Check 约束）则弹出气泡报错并回滚单元格内容。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Playwright E2E 测试脚本）**：
```typescript
test('inline cell editing persists change and handles validation error', async ({ page }) => {
  await page.goto('/project/default/editor');
  await page.click('button:has-text("customers")'); // 切换到 customers 数据表

  // 1. 定位第 1 行的 email 单元格
  const emailCell = page.locator('[role="gridcell"][data-col-id="email"]').first();
  await emailCell.dblclick(); // 双击激活 Monaco/Input 编辑器

  // 2. 模拟用户输入新邮箱并回车
  const editorInput = page.locator('.cell-editor-input');
  await editorInput.fill('updated_customer@example.com');
  await editorInput.press('Enter');

  // 3. 拦截向 PostgREST 发出的真实更新请求
  const patchResponse = await page.waitForResponse(res => 
    res.url().includes('/rest/v1/customers') && res.request().method() === 'PATCH'
  );
  expect(patchResponse.status()).toBe(204); // Postgres 更新成功返回 204 No Content

  // 4. 验证单元格文本已正确同步显示
  await expect(emailCell).toHaveText('updated_customer@example.com');

  // 5. 负向边界测试：输入重复邮箱触发唯一键冲突
  await emailCell.dblclick();
  await editorInput.fill('already_existing@example.com');
  await editorInput.press('Enter');

  // 6. 断言错误气泡弹出并展示 Postgres 物理错误提示
  await expect(page.locator('.toast-error')).toContainText('violates unique constraint');
});
```

---

### 3.2 功能二：可视化 RLS 行级安全策略装配与多角色断言

#### 3.2.1 业务功能背景
管理员在 UI 上为 `todos` 表勾选“Enable Row Level Security”，并使用策略向导添加模板：“Users can only see their own todos”，表达式为 `auth.uid() = user_id`。

#### 3.2.2 测试用例设计方案
* **设计策略**：
  1. 通过 Playwright 驱动 UI 完成策略配置并点击 Save。
  2. 测试程序直接生成两枚合法的 JWT Token（`User-A` 与 `User-B`）。
  3. 分别携带两个 Token 向 `/rest/v1/todos` 发起 GET 请求。
  4. **硬性断言**：
     * `User-A` 返回的结果集包含且仅包含属于 `User-A` 的记录。
     * `User-B` 绝对无法读取 `User-A` 的数据行，物理验证 RLS 在数据库内核生效。

---

### 3.3 功能三：数据库内部契约断言（pgTAP 自动化用例）

#### 3.3.1 业务功能背景
如何证明每次系统迁移脚本（Migrations）执行后，所有的外键、索引、触发器与默认值都处于预期的健康状态？

#### 3.3.2 测试用例设计方案
* 在 PostgreSQL 内部直接运行 `pgTAP` 单元测试脚本：
```sql
BEGIN;
SELECT plan(4);

-- 1. 断言表是否存在
SELECT has_table('customers', 'customers table should exist');

-- 2. 断言列与数据类型
SELECT has_column('customers', 'email', 'customers table should have email column');
SELECT col_type_is('customers', 'email', 'text', 'email column should be text');

-- 3. 断言外键约束与 RLS 开启状态
SELECT table_has_fk('orders', 'orders table must have foreign keys');

SELECT * FROM finish();
ROLLBACK;
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Supabase Local CLI 本地栈沙箱**：
   * 所有测试依赖 `supabase start` 在本地无缝拉起由 8 个官方镜像组成的完整 PostgreSQL 环境，支持秒级创建与销毁。
2. **数据库重置与种子填充（Seed Data）自动化**：
   * 测试套件前置执行 `supabase db reset`，自动应用所有历史 Migration 并在毫秒内填充初始种子数据，保证各 CI 运行器状态绝对归一。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **网格/表格类复杂 Web 系统必须设计“失焦与键盘事件测试”**：在设计包含复杂输入网格的系统时，回车提交、Tab 键切换下一格、Esc 取消、点击外部失焦，是 Bug 频发的重灾区，必须纳入端到端重点用例。
* **深层后端配置操作必须执行“数据面验证闭环”**：在管理控制台（Control Plane）上配置了安全策略、路由或权限后，不能仅断言控制台提示“成功”，必须立即模拟真实终端用户从数据面（Data Plane）发起请求检验效果。
