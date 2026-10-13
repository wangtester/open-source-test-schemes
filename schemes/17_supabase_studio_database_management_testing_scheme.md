# Supabase Studio 自动化测试方案与全量用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Supabase Studio](https://github.com/supabase/supabase) 是开源云原生数据库工作台的业界标杆，采用 Next.js (App Router)、React 19、Tailwind CSS、Monaco Editor、React Flow 与 Data Grid 虚拟化表格构建。其后端直接深度连接 PostgreSQL 内核，并通过 PostgREST、GoTrue（Auth）、Realtime（WAL 逻辑复制流广播）以及 Storage API 提供全栈数据管理服务。

### 1.2 被测系统核心组成与难点
1. **Table Editor（虚拟化表格编辑器）**：支持百万级行数据虚拟化滚动、动态列定义、行级/单元格就地编辑（Inline Edit）、外键跳转。
2. **SQL Editor（交互式查询控制台）**：多标签页 Monaco 编辑器、SQL 历史记录、参数化执行、慢查询取消（`pg_cancel_backend`）。
3. **Policies & RLS Builder（行级安全可视化设计器）**：针对不同数据表与操作（SELECT/INSERT/UPDATE/DELETE）装配安全表达式，直接操作 Postgres 系统表 `pg_policy`。
4. **Database Advanced Tools（数据库扩展与函数）**：可视化启停 `pgvector`/`postgis` 扩展、编写触发器与存储过程、绘制 ERD 实体关系图。
5. **Storage & Edge Functions（存储与函数管理）**：存储桶管理、文件分块断点续传、边缘函数 Secrets 配置与日志追踪。

---

## 2. 自动化测试分层架构与环境编排

Supabase Studio 采用了从前端无头浏览器到数据库物理内核的纵深防御测试架构：

```
┌────────────────────────────────────────────────────────┐
│             Playwright Studio Web E2E Suite            │  顶层 UI 交互：全页面表单、虚拟网格拖拽、Monaco 编辑器
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         PostgREST & GoTrue API Contract Integration    │  接口契约层：验证前端操作下发时 REST API 的参数与状态码
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      pgTAP In-Database SQL Assertion Suite             │  数据库内核层：直接在 Postgres 内部运行 SQL 断言函数
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Supabase Local CLI Docker Environment          │  基础设施底座：包含 Postgres, Kong, PostgREST 完整容器栈
└────────────────────────────────────────────────────────┘
```

---

## 3. 全量核心功能自动化测试用例设计与代码实现

### 3.1 模块一：Table Editor 数据表全生命周期管理

#### 3.1.1 功能 1：动态创建数据表（包含主键、RLS 与 Realtime 配置）
* **测试场景**：用户在 Studio 界面点击“New Table”，输入表名 `orders`，勾选“Enable Row Level Security”与“Enable Realtime”，添加主键 `id (uuid, default: gen_random_uuid())`。
* **驱动方式**：Playwright 表单交互。
* **断言设计**：
  1. 前端 Toast 提示成功；
  2. 数据库直接查询 `information_schema.tables` 验证物理表已存在；
  3. 查询 `pg_class.relrowsecurity` 验证 RLS 标志为 `true`；
  4. 查询 `pg_publication_tables` 验证该表已加入 `supabase_realtime` 逻辑复制发布。
* **完整测试代码**：
```typescript
test('create table with RLS and Realtime enabled', async ({ page }) => {
  await page.goto('/project/default/editor');
  await page.click('button:has-text("New table")');

  // 1. 填写表名与基础配置
  await page.fill('input[placeholder="table_name"]', 'orders');
  await page.check('input[name="enable-rls"]');
  await page.check('input[name="enable-realtime"]');

  // 2. 配置主键列为 UUID 且带默认值
  const primaryKeyType = page.locator('button[aria-label="Select column type"]').first();
  await primaryKeyType.click();
  await page.fill('input[placeholder="Search types..."]', 'uuid');
  await page.click('div[role="option"]:has-text("uuid")');
  await page.fill('input[placeholder="DEFAULT"]', 'gen_random_uuid()');

  // 3. 提交创建表
  await page.click('button:has-text("Save")');
  await expect(page.locator('.toast-success')).toBeVisible();

  // 4. 物理世界断言：直接执行 SQL 验证 PostgreSQL 底层系统元数据
  const meta = await executeSql(`
    SELECT c.relrowsecurity, 
           EXISTS(SELECT 1 FROM pg_publication_tables WHERE tablename = 'orders' AND pubname = 'supabase_realtime') as in_realtime
    FROM pg_class c 
    JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE c.relname = 'orders' AND n.nspname = 'public';
  `);
  expect(meta.rows[0].relrowsecurity).toBe(true);
  expect(meta.rows[0].in_realtime).toBe(true);
});
```

#### 3.1.2 功能 2：修改列结构（添加外键约束与非空约束）
* **测试场景**：为已有表 `orders` 添加一列 `customer_id`，类型为 `uuid`，外键引用 `customers.id`，级联删除规则设置为 `CASCADE`。
* **完整测试代码**：
```typescript
test('add column with foreign key cascade constraint', async ({ page }) => {
  await page.goto('/project/default/editor/orders');
  await page.click('button:has-text("Insert column")');

  // 填写列名与类型
  await page.fill('input[name="column-name"]', 'customer_id');
  await selectColumnType(page, 'uuid');

  // 展开外键关系设置
  await page.click('button:has-text("Add foreign key relation")');
  await page.selectOption('select[name="foreign-table"]', 'customers');
  await page.selectOption('select[name="foreign-column"]', 'id');
  await page.selectOption('select[name="on-delete"]', 'Cascade');

  await page.click('button:has-text("Save column")');
  await expect(page.locator('.toast-success')).toBeVisible();

  // 物理断言：校验 Postgres 外键约束
  const fkCheck = await executeSql(`
    SELECT confdeltype FROM pg_constraint 
    WHERE conname = 'orders_customer_id_fkey' AND contype = 'f';
  `);
  expect(fkCheck.rows[0].confdeltype).toBe('c'); // 'c' 代表 CASCADE
});
```

---

### 3.2 模块二：虚拟化网格数据增删改（Data Grid Virtual Scrolling）

#### 3.2.1 功能 1：虚拟滚动长列表单元格就地编辑（Inline Cell Editing）与失焦持久化
* **测试场景**：在包含 10,000 条数据的表格中，向下滚动至第 500 行，双击 `status` 单元格，修改为 `completed`，按 Enter 键保存。
* **完整测试代码**：
```typescript
test('virtual scroll down and inline edit cell with blur save', async ({ page }) => {
  await page.goto('/project/default/editor/orders');

  // 1. 模拟虚拟滚动至第 500 行区域
  const gridContainer = page.locator('.rdg');
  await gridContainer.evaluate((el) => el.scrollTop = 15000); // 触发虚拟 DOM 重绘
  await page.waitForTimeout(300); // 等待新视口行加载

  // 2. 定位视口内出现的行单元格
  const targetCell = page.locator('.rdg-cell[data-col-key="status"]').first();
  await targetCell.dblclick();

  // 3. 输入新值并触发提交
  const editorInput = page.locator('.rdg-text-editor');
  await editorInput.fill('completed');
  await editorInput.press('Enter');

  // 4. 拦截向 PostgREST 发送的 PATCH 请求并断言状态码
  const patchReq = await page.waitForResponse(
    (res) => res.url().includes('/rest/v1/orders') && res.request().method() === 'PATCH'
  );
  expect(patchReq.status()).toBe(204);

  // 5. 校验单元格文本已正确同步
  await expect(targetCell).toHaveText('completed');
});
```

#### 3.2.2 功能 2：违反 Check 约束或唯一键冲突的错误回滚
* **测试场景**：修改用户邮箱单元格为已存在的重复邮箱，断言弹出 Toast 错误，单元格内容自动回退为旧值。
* **完整测试代码**：
```typescript
test('cell editing uniqueness violation rolls back to previous value', async ({ page }) => {
  await page.goto('/project/default/editor/customers');
  const emailCell = page.locator('.rdg-cell[data-col-key="email"]').first();
  const originalEmail = await emailCell.textContent();

  await emailCell.dblclick();
  await page.locator('.rdg-text-editor').fill('duplicate@example.com');
  await page.keyboard.press('Enter');

  // 断言错误提示
  await expect(page.locator('.toast-error')).toContainText('violates unique constraint');
  // 断言单元格内容未被污染，保持原始值
  await expect(emailCell).toHaveText(originalEmail!);
});
```

#### 3.2.3 功能 3：多条件复合筛选（Filter）与排序（Sort）
* **测试场景**：添加筛选 `total > 100` 且 `status == 'pending'`，按 `created_at DESC` 排序。
* **完整测试代码**：
```typescript
test('apply complex multi-condition filter and sort', async ({ page }) => {
  await page.goto('/project/default/editor/orders');

  // 点击 Filter 按钮并添加条件
  await page.click('button:has-text("Filter")');
  await page.click('button:has-text("Add filter")');
  await page.selectOption('select[data-testid="filter-column"]', 'total');
  await page.selectOption('select[data-testid="filter-operator"]', 'gt');
  await page.fill('input[data-testid="filter-value"]', '100');

  await page.click('button:has-text("Add filter")');
  await page.locator('select[data-testid="filter-column"]').nth(1).selectOption('status');
  await page.locator('select[data-testid="filter-operator"]').nth(1).selectOption('eq');
  await page.locator('input[data-testid="filter-value"]').nth(1).fill('pending');

  await page.click('button:has-text("Apply filter")');

  // 拦截网络请求，断言 PostgREST 查询参数格式
  const req = await page.waitForRequest((r) => r.url().includes('/rest/v1/orders'));
  expect(req.url()).toContain('total=gt.100');
  expect(req.url()).toContain('status=eq.pending');
});
```

---

### 3.3 模块三：SQL Editor 查询控制台与慢查询中断

#### 3.3.1 功能 1：多标签页 Monaco 编辑器执行 SQL 与结果渲染
* **测试场景**：在 SQL Editor 键入 `SELECT * FROM customers WHERE active = true;`，快捷键 Ctrl+Enter 运行，验证下方结果表格列数与行数。
* **完整测试代码**：
```typescript
test('execute SQL in Monaco editor and view formatted result grid', async ({ page }) => {
  await page.goto('/project/default/sql/new');

  // 1. 在 Monaco Editor 注入 SQL
  await page.click('.monaco-editor');
  await page.keyboard.insertText('SELECT name, email, active FROM customers LIMIT 5;');

  // 2. 模拟快捷键 Ctrl+Enter (Mac 上为 Cmd+Enter)
  await page.keyboard.press(process.platform === 'darwin' ? 'Meta+Enter' : 'Control+Enter');

  // 3. 断言执行耗时与行数提示
  await expect(page.locator('.sql-run-status')).toContainText('Success');
  await expect(page.locator('.sql-run-status')).toContainText('5 rows');

  // 4. 断言结果数据表表头
  const headers = page.locator('.sql-result-table th');
  await expect(headers).toHaveText(['name', 'email', 'active']);
});
```

#### 3.3.2 功能 2：长时间运行慢查询主动取消（Query Cancellation）
* **测试场景**：执行耗时 30 秒的慢查询 `SELECT pg_sleep(30);`，在运行状态中点击“Cancel query”按钮，断言查询在 500ms 内被终止，后端释放连接。
* **完整测试代码**：
```typescript
test('cancel running long query terminates backend process cleanly', async ({ page }) => {
  await page.goto('/project/default/sql/new');
  await page.click('.monaco-editor');
  await page.keyboard.insertText('SELECT pg_sleep(30);');

  // 触发运行
  await page.click('button:has-text("Run")');
  await expect(page.locator('button:has-text("Cancel")')).toBeVisible();

  // 点击取消
  await page.click('button:has-text("Cancel")');

  // 断言前端捕获用户取消错误提示
  await expect(page.locator('.sql-run-status')).toContainText('canceling statement due to user request');

  // 物理断言：查询 pg_stat_activity 验证后台没有挂起的 sleep 进程
  const checkActive = await executeSql(`
    SELECT count(*) FROM pg_stat_activity 
    WHERE query LIKE '%pg_sleep(30)%' AND state = 'active';
  `);
  expect(parseInt(checkActive.rows[0].count, 10)).toBe(0);
});
```

---

### 3.4 模块四：Policies & RLS 行级安全策略可视化构建器

#### 3.4.1 功能 1：为特定操作（INSERT / SELECT）装配细粒度安全规则
* **测试场景**：为 `documents` 表创建一条针对 `authenticated` 角色的 `INSERT` 策略，`WITH CHECK` 表达式为 `auth.uid() = owner_id`。
* **完整测试代码**：
```typescript
test('create granular RLS policy with WITH CHECK expression', async ({ page }) => {
  await page.goto('/project/default/auth/policies');
  await page.click('button[data-testid="new-policy-documents"]');
  await page.click('button:has-text("For full customization")');

  // 1. 填写策略名称与操作类型
  await page.fill('input[name="policy-name"]', 'Users can only insert their own documents');
  await page.selectOption('select[name="allowed-operation"]', 'INSERT');
  await page.selectOption('select[name="target-roles"]', 'authenticated');

  // 2. 填写 WITH CHECK 表达式
  await page.fill('textarea[name="with-check-expression"]', 'auth.uid() = owner_id');

  await page.click('button:has-text("Save policy")');
  await expect(page.locator('.toast-success')).toBeVisible();

  // 3. 物理断言：查询 pg_policy 系统表
  const pol = await executeSql(`
    SELECT polcmd, polroles::regrole[] as roles, pg_get_expr(polwithcheck, polrelid) as with_check
    FROM pg_policy 
    WHERE polname = 'Users can only insert their own documents';
  `);
  expect(pol.rows[0].polcmd).toBe('a'); // 'a' 代表 INSERT
  expect(pol.rows[0].with_check).toContain('auth.uid() = owner_id');
});
```

#### 3.4.2 功能 2：双客户端 JWT 身份验证行级数据隔离有效性断言
* **测试场景**：Alice 插入记录 `owner_id = 'alice-uuid'`；Bob 携带自身 JWT 请求，断言绝对无法通过 SELECT 查询到 Alice 的记录。
* **完整测试代码**：
```typescript
test('RLS policy physically enforces data isolation across two distinct JWTs', async () => {
  const aliceToken = generateJwt({ sub: 'alice-uuid', role: 'authenticated' });
  const bobToken = generateJwt({ sub: 'bob-uuid', role: 'authenticated' });

  // 1. Alice 写入属于自己的文档
  const insertRes = await fetch(`${SUPABASE_URL}/rest/v1/documents`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${aliceToken}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({ title: 'Alice Secret Document', owner_id: 'alice-uuid' }),
  });
  expect(insertRes.status).toBe(201);

  // 2. Bob 发起查询
  const bobRes = await fetch(`${SUPABASE_URL}/rest/v1/documents`, {
    headers: { Authorization: `Bearer ${bobToken}` },
  });
  const bobData = await bobRes.json();

  // 严格断言：Bob 获取的结果集中绝对不包含 Alice 的记录
  expect(bobData).toHaveLength(0);
});
```

---

### 3.5 模块五：数据库扩展（Extensions）与高级元数据管理

#### 3.5.1 功能 1：一键开启与禁用 PostgreSQL 扩展（如 pgvector）
* **测试场景**：在 Database Extensions 页面搜索 `vector`，点击开关启用扩展；断言执行 `CREATE EXTENSION vector`。
* **完整测试代码**：
```typescript
test('toggle pgvector extension and verify database availability', async ({ page }) => {
  await page.goto('/project/default/database/extensions');
  await page.fill('input[placeholder="Search extensions..."]', 'vector');

  // 点击启用开关
  const vectorToggle = page.locator('button[data-testid="extension-toggle-vector"]');
  await vectorToggle.click();
  await page.click('button:has-text("Confirm install")');

  await expect(page.locator('.toast-success')).toBeVisible();

  // 物理断言：校验 pg_extension 系统表
  const extCheck = await executeSql("SELECT extversion FROM pg_extension WHERE extname = 'vector';");
  expect(extCheck.rows.length).toBe(1);

  // 验证 vector 类型可以在后续创建表时被正常使用
  await executeSql("CREATE TABLE test_embeddings (id serial, vec vector(3)); DROP TABLE test_embeddings;");
});
```

#### 3.5.2 功能 2：数据库函数与触发器（Triggers）绑定闭环测试
* **测试场景**：在界面创建函数 `update_timestamp()`，绑定为在 `orders` 表更新时的 `BEFORE UPDATE` 触发器。
* **完整测试代码**：
```typescript
test('create database function and bind trigger to table', async ({ page }) => {
  await page.goto('/project/default/database/triggers');
  await page.click('button:has-text("Create trigger")');

  await page.fill('input[name="trigger-name"]', 'set_updated_at');
  await page.selectOption('select[name="table"]', 'orders');
  await page.selectOption('select[name="events"]', 'UPDATE');
  await page.selectOption('select[name="orientation"]', 'ROW');
  await page.selectOption('select[name="function"]', 'update_timestamp');

  await page.click('button:has-text("Save")');
  await expect(page.locator('.toast-success')).toBeVisible();

  // 物理断言：触发器存在性
  const trig = await executeSql("SELECT tgname FROM pg_trigger WHERE tgname = 'set_updated_at';");
  expect(trig.rows.length).toBe(1);
});
```

---

### 3.6 模块六：Storage 存储桶管理与安全策略

#### 3.6.1 功能 1：私有存储桶创建与文件拖拽上传
* **测试场景**：创建名为 `avatars` 的私有存储桶，拖入一张 PNG 图片，断言文件列表中呈现该文件及其尺寸。
* **完整测试代码**：
```typescript
test('create private bucket and upload file via drag and drop', async ({ page }) => {
  await page.goto('/project/default/storage/buckets');
  await page.click('button:has-text("New bucket")');

  await page.fill('input[name="bucket-name"]', 'avatars');
  // 保持私有桶（Private）不勾选 Public
  await page.click('button:has-text("Create bucket")');
  await expect(page.locator('.toast-success')).toBeVisible();

  // 进入桶并模拟文件上传
  await page.click('a:has-text("avatars")');
  const fileChooserPromise = page.waitForEvent('filechooser');
  await page.click('button:has-text("Upload file")');
  const fileChooser = await fileChooserPromise;
  await fileChooser.setFiles({
    name: 'avatar.png',
    mimeType: 'image/png',
    buffer: Buffer.from('fake_image_content'),
  });

  // 断言文件列表中出现该条目
  await expect(page.locator('.storage-file-row:has-text("avatar.png")')).toBeVisible();
});
```

---

### 3.7 模块七：Edge Functions 与环境变量配置

#### 3.7.1 功能 1：边缘函数机密环境变量（Secrets）加密增删与脱敏校验
* **测试场景**：为项目添加密钥 `STRIPE_KEY = sk_test_12345`，断言添加后在列表中以掩码显示 `••••••••`，无法直接明文读取。
* **完整测试代码**：
```typescript
test('manage edge function secret with masked UI presentation', async ({ page }) => {
  await page.goto('/project/default/functions/secrets');
  await page.click('button:has-text("Add new secret")');

  await page.fill('input[placeholder="SECRET_NAME"]', 'STRIPE_KEY');
  await page.fill('input[placeholder="Secret value"]', 'sk_test_12345');
  await page.click('button:has-text("Save")');

  await expect(page.locator('.toast-success')).toBeVisible();

  // 视图断言：值必须脱敏
  const secretRow = page.locator('.secret-row:has-text("STRIPE_KEY")');
  await expect(secretRow.locator('.secret-value')).toHaveText('••••••••••••');
});
```

---

### 3.8 模块八：可观察性与日志分析（Logs Explorer）

#### 3.8.1 功能 1：Postgres 查询慢日志过滤与错误堆栈定位
* **测试场景**：执行一条语法错误的 SQL 后，进入 Logs Explorer 页面，输入过滤条件 `event_message ~ "syntax error"`，断言在 2 秒内查询到对应的错误日志行。
* **完整测试代码**：
```typescript
test('logs explorer captures postgres syntax error log entry', async ({ page }) => {
  // 故意制造一条错误 SQL
  await executeSql("SELECT invalid_column_xyz FROM non_existing_table_abc;").catch(() => {});

  await page.goto('/project/default/logs/explorer');
  await page.click('button:has-text("Postgres logs")');
  await page.fill('input[placeholder="Search logs..."]', 'invalid_column_xyz');
  await page.press('input[placeholder="Search logs..."]', 'Enter');

  // 断言日志流中显示出对应的错误行
  await expect(page.locator('.log-event-row:has-text("invalid_column_xyz")').first()).toBeVisible();
});
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Supabase Local CLI 本地多容器拓扑沙箱**：
   * 所有测试在执行前自动调用 `supabase start`，启动包含 PostgreSQL 15, PostgREST 12, GoTrue, Realtime, Kong, Storage 的完整微服务环境。
2. **测试数据库原子复位机制（Fast Reset）**：
   * 在测试套件前后，调用 `supabase db reset` 执行全量 Migrations 并填充初始 Seed 数据，保证用例执行的零状态污染。
3. **pgTAP 数据库内核测试门禁**：
   * 除了前端 Playwright 外，每次提交运行 `pg_prove` 执行底层的纯 SQL 单元测试，拦截任何数据库函数与模式的破坏性改动。

---

## 5. 对复杂 Web 数据库系统测试设计的实战借鉴意义

* **复杂数据系统测试必须实现“前台交互 + 后台系统表”双重校验**：在网页端修改了表结构、外键、约束、安全策略后，绝不能仅断言前端弹出“保存成功”，必须直接通过底层驱动查询系统目录（如 `pg_class`, `pg_policy`, `pg_constraint`），以物理元数据为唯一黄金标准。
* **虚拟化网格组件必须编写“滚动位移与键盘事件测试”**：对于承载数万行的大数据表格，传统的 DOM 定位会全部失效；必须结合虚拟滚动高度计算（`scrollTop`）、单元格双击激活编辑器、Tab/Enter 快捷键与失焦事件，进行高精度交互模拟。
* **长耗时操作必须打通“取消链路（Cancellation）”的闭环测试**：无论是大文件上传、慢 SQL 执行，还是批处理任务，测试必须涵盖主动中断（Cancel）的场景，验证连接池释放与无脏数据残留。
