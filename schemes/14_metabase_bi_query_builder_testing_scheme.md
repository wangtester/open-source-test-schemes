# Metabase 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[Metabase](https://github.com/metabase/metabase) 是全球最流行的开源商业智能（BI）与数据可视化分析平台。前端采用 React + Redux + D3 图表引擎，后端采用 Clojure 编写的高吞吐分析引擎。系统支持通过可视化点击（Query Builder / Notebook Editor）无代码生成多表 JOIN、聚合计算，同时兼容原生 SQL 编辑，内置强大的权限沙盒（Row-Level Data Sandboxing）。

```
┌────────────────────────────────────────────────────────────────────────┐
│             Metabase Frontend (React + Redux + Query Builder)          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ MBQL (Metabase Query Language JSON)
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Metabase Core Backend (Clojure)                     │
├───────────────────────────────┬────────────────────────────────────────┤
│ MBQL Query Compiler Engine    │ Data Sandboxing & Permission Engine    │
│ - MBQL AST 递归宏展开         │ - 用户属性动态注入 WHERE 过滤条件      │
│ - 多数据库方言 SQL 转译器    │ - 敏感字段列级屏蔽 (Column Masking)    │
├───────────────────────────────┼────────────────────────────────────────┤
│ Driver Adapter Pipeline       │ Query Cache & Execution Manager        │
│ - Postgres / BigQuery / MySQL │ - 相同查询哈希缓存与指纹计算           │
│ - 异步流式返回 Dataset Rows   │ - 查询取消与客户端长连接断开检测       │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Metabase 拥有在 Clojure 与 React 生态中最严苛的自动化测试规范之一：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **MBQL Compiler Unit** | AST 语法树生成、隐式 JOIN 补全、时间维度截断（`bucket: :month`） | Clojure `clojure.test` | 纯内存符号转换，无外部数据库 |
| **Driver Integration Tests** | 编译后的 SQL 在真实数据库（Postgres, MySQL, Snowflake 等）执行 | Testcontainers / Docker | 统一标准数据集（`test-data`）初始化 |
| **Data Sandboxing Tests** | 针对普通用户、受限用户自动附加行级过滤条件（WHERE user_id = x） | Clojure 测试框架 | 模拟不同 Session Token 鉴权上下文 |
| **Frontend Notebook Tests** | Notebook 模式下的添加 Filter、添加 Summarize、多表拖拽关联 | Jest + RTL + Cypress / Playwright | 拦截 MBQL API 响应数据 |
| **Visual Regression Tests** | 折线图、透视表（Pivot Table）、漏斗图渲染在各种分辨率下的像素一致性 | Playwright Visual Matcher | 冻结动画，统一字体与抗锯齿渲染 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：MBQL（Metabase 查询语言）编译与多表隐式 JOIN 转译测试

#### 3.1.1 业务功能与测试目标
当用户在界面上选择“订单表（Orders）”并按“产品类别（Products.Category）”聚合统计时，MBQL 内部生成如下数据结构：
`{:database 1, :query {:source-table 10, :breakout [[:field 25 {:source-field 15}]]}}`
编译器必须自动识别 `Orders.product_id -> Products.id` 的外键关系，并在最终生成的 SQL 中补上 `LEFT JOIN products ON orders.product_id = products.id`。

#### 3.1.2 测试用例实现
```clojure
(ns metabase.query-processor-test
  (:require [clojure.test :refer :all]
            [metabase.query-processor :as qp]
            [metabase.test :as mt]))

(deftest test-mbql-implicit-join-compilation
  (mt/test-driver :postgres
    (testing "MBQL automatically generates LEFT JOIN for foreign key breakouts"
      ;; 1. 构造一个包含跨表外键引用的 MBQL 查询
      (let [query (mt/mbql-query orders
                    {:aggregation [[:count]]
                     :breakout    [!default.products.category]})
            ;; 2. 调用查询编译管道将其编译为 Postgres 原生 SQL
            compiled-sql (qp/compile query)]
        
        ;; 3. 结构化严格断言：必须生成正确的 JOIN 和 GROUP BY
        (is (= true (contains? compiled-sql :query)))
        (is (re-find #"LEFT JOIN \"public\"\.\"products\"" (:query compiled-sql)))
        (is (re-find #"ON \"public\"\.\"orders\"\.\"product_id\" = \"public\"\.\"products\"\.\"id\"" (:query compiled-sql)))
        (is (re-find #"GROUP BY \"products__via__product_id\"\.\"category\"" (:query compiled-sql)))))))
```

---

### 3.2 模块二：数据沙盒（Data Sandboxing）行级权限注入安全测试

#### 3.2.1 业务功能与测试目标
在企业版中，普通部门员工只能查阅本部门数据。测试必须验证：
1. 即使恶意用户通过修改前端请求，试图在 MBQL 中发起全表扫描（`SELECT * FROM orders`）。
2. 后端中间件强制拦截并自动重写查询 AST，追加 `WHERE department = 'HR'`。
3. 任何试图绕过沙盒的非法注入参数被强行覆写。

#### 3.2.2 测试用例实现
```clojure
(ns metabase.sandboxing-test
  (:require [clojure.test :refer :all]
            [metabase.query-processor :as qp]
            [metabase.test :as mt]))

(deftest test-data-sandboxing-appends-row-level-filter
  (mt/with-test-user :sandboxed-user
    (testing "Sandboxed user query is strictly scoped by assigned department attribute"
      ;; 构造试图查阅全部客户的查询
      (let [malicious-query (mt/mbql-query customers)
            result (qp/process-query malicious-query)]
        
        ;; 1. 断言返回的数据行全部属于该受限制用户的部门
        (is (every? #(= "Engineering" (nth % 2)) (mt/rows result)))

        ;; 2. 检查底层实际执行的 SQL 确实追加了参数化限定条件
        (let [compiled (qp/compile malicious-query)]
          (is (re-find #"WHERE \"public\"\.\"customers\"\.\"department\" = \?" (:query compiled)))
          (is (= ["Engineering"] (:params compiled))))))))
```

---

### 3.3 模块三：Query Caching 机制与指纹（Fingerprint）失效测试

#### 3.3.1 业务功能与测试目标
相同 MBQL 查询必须优先命中缓存。测试必须覆盖：
1. 首次查询物理访问数据库并落入 Redis/内存缓存。
2. 二次查询完全命中缓存，响应时间小于 5ms，且底层数据库执行计数不变。
3. 当用户更新仪表盘卡片定义（如改变图表过滤参数）时，老缓存自动失效。

#### 3.3.2 测试用例实现
```clojure
(ns metabase.caching-test
  (:require [clojure.test :refer :all]
            [metabase.query-processor.cache :as cache]
            [metabase.test :as mt]))

(deftest test-query-caching-lifecycle
  (let [query (mt/mbql-query orders {:limit 10})
        cache-hash (cache/query-hash query)]
    ;; 确保初始缓存为空
    (cache/evict! cache-hash)
    (is (nil? (cache/get-cache-entry cache-hash)))

    ;; 1. 首次执行查询，验证写入缓存
    (let [first-run (mt/run-query-with-cache query)]
      (is (= false (:cached? first-run)))
      (is (some? (cache/get-cache-entry cache-hash))))

    ;; 2. 二次执行相同查询，验证命中缓存
    (let [second-run (mt/run-query-with-cache query)]
      (is (= true (:cached? second-run)))
      (is (= (:rows first-run) (:rows second-run))))))
```

---

### 3.4 模块四：Playwright Notebook 编辑器可视化多表 Join 与透视表 E2E 测试

#### 3.4.1 业务功能与测试目标
驱动浏览器验证数据分析师的核心操作路径：
1. 点击“Ask a question” -> 选择 Notebook 模式。
2. 关联第二张数据表并配置聚合列。
3. 点击“Visualize”进入图表视图并切换为透视表（Pivot Table）。

#### 3.4.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('Metabase Notebook Query Builder E2E', () => {
  test('User builds multi-table join query and renders pivot table', async ({ page }) => {
    await page.goto('http://localhost:3000/question/new');

    // 1. 选择初始表：Orders
    await page.click('button:has-text("Raw data")');
    await page.click('text=Sample Database');
    await page.click('text=Orders');

    // 2. 在 Notebook 中点击 Join 数据表按钮
    await page.click('[aria-label="Join data"]');
    await page.click('text=Products');

    // 3. 添加 Summarize 聚合指标：Count of rows by Category
    await page.click('[aria-label="Summarize"]');
    await page.click('text=Pick a column to group by');
    await page.click('text=Category');

    // 4. 点击运行查询按钮
    await page.click('button:has-text("Visualize")');

    // 5. 校验数据表格渲染出两列：Category 与 Count
    const dataTable = page.locator('.QueryTable');
    await expect(dataTable).toBeVisible({ timeout: 15000 });
    await expect(dataTable).toContainText('Doohickey');
    await expect(dataTable).toContainText('Gadget');

    // 6. 切换展示模式为 Pivot Table 透视表
    await page.click('button:has-text("Visualization")');
    await page.click('[data-testid="pivot-table-button"]');
    await expect(page.locator('.PivotTable')).toBeVisible();
  });
});
```

---

## 4. 测试工程化与基础设施

1. **统一标准测试数据集（`test-data`）**：
   * Metabase 为所有测试编写了一套确定性的测试数据库（包含 Orders、Products、Users、Reviews 表及固定种子数据），全套单测与集成测试均基于此固定状态集，彻底杜绝数据漂移。
2. **多方言 Driver 抽象测试矩阵**：
   * 采用统一的 `mt/test-driver` 宏，只需声明一段测试逻辑，测试框架自动循环在 Postgres、MySQL、H2、SQLite 上复现执行。

---

## 5. 对同类复杂系统的参考价值

* **数据中台系统的 AST 级拦截与沙箱安全断言**：在 BI 和低代码系统设计中，不能指望前端传来的 SQL 是可信的。将用户交互转为中间表示（如 MBQL AST），并在编译层统一动态织入安全策略，配合全面的 AST 单元断言，是构建金融级安全系统的基石。
