# Metabase 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Metabase](https://github.com/metabase/metabase) 是全球最流行的开源企业级商业智能（BI）与数据可视化分析平台。系统后端采用 Clojure 编写核心数据处理与抽象查询语言（MBQL）转换引擎，前端采用 React、Redux 与 D3 图表库构建交互式报表，能够无缝连接 PostgreSQL、MySQL、ClickHouse、Snowflake、BigQuery 等数十种数据库。

### 1.2 被测系统核心组成
1. **可视化查询构建器（Visual Query Builder）**：非技术人员通过点击即可组合筛选（Filter）、聚合（Summarize）、分组（Group By）以及多表关联（Join）。
2. **MBQL 统一抽象查询引擎**：将前端生成的 JSON AST 查询树转译为对应目标数据库的物理 SQL 方言。
3. **数据权限沙箱（Data Sandboxing & Row-Level Security）**：根据登录用户的部门/角色，在运行时向底层查询强行拼接数据行级过滤条件。
4. **图表钻取与联动过滤（Drill-through & Dashboard Cross-filtering）**：点击柱状图某根柱子，整张看板所有相关图表自动按该维度下钻联动。

### 1.3 自动化测试的主要难点与挑战
* **前端高度复杂的动态表单交互与即时 SQL 生成**：任意筛选条件的增删都会导致前端状态树变化，需验证复杂表单交互不会产生语法损坏的查询。
* **多异构数据库方言（Dialect）兼容性**：同一种时间聚合（如“按季度提取”），PostgreSQL 与 MySQL、ClickHouse 的函数实现完全不同。
* **长时间运行查询的取消与连接释放**：用户在页面点击“取消查询”时，必须保证后端真正终止数据库端的执行线程，防止连接池泄漏。

---

## 2. 测试架构全景与分层策略

Metabase 的自动化测试架构分为前端交互驱动与后端多数据库转译两套重型防护网：

```
┌────────────────────────────────────────────────────────┐
│             Cypress Comprehensive E2E Matrix           │  前端集成层：驱动可视化 Query Builder、看板联动与导出
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         MBQL Query Processor Clojure Tests             │  转译计算层：验证 JSON AST 转译为目标物理 SQL 的正确性
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│       Driver Integration Tests (Dockerized DBs)        │  驱动验证层：针对真实 Postgres/MySQL 容器执行查询并比对数据
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Permission & Sandboxing Security Suite         │  安全沙箱层：多角色登录验证数据行/列级权限隔离
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：可视化 Query Builder 交互与动态多表关联（Join）

#### 3.1.1 业务功能背景
用户选择“Orders”表，点击“Join Data”关联“Products”表，配置关联字段 `Orders.Product_ID = Products.ID`，再添加聚合指标 `Sum(Orders.Total)` 按 `Products.Category` 分组，验证页面能够实时渲染出正确的统计透视表。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Cypress E2E 测试脚本）**：
```javascript
describe("Query Builder: Complex Join and Summarize", () => {
  it("should create a join, summarize by foreign category and display table", () => {
    cy.visit("/question/new");
    cy.findByText("Raw data").click();
    cy.findByText("Sample Database").click();
    cy.findByText("Orders").click();

    // 1. 点击 Join 按钮添加外键关联表
    cy.findByTestId("join-data-button").click();
    cy.findByText("Products").click();

    // 2. 添加聚合计算：Sum of Total
    cy.findByTestId("summarize-button").click();
    cy.findByText("Sum of ...").click();
    cy.findByText("Total").click();

    // 3. 添加分组维度：Products -> Category
    cy.findByText("Group by").click();
    cy.findByText("Products").click();
    cy.findByText("Category").click();

    // 4. 点击运行并断言网络返回与表格呈现
    cy.intercept("POST", "/api/dataset").as("datasetQuery");
    cy.findByTestId("visualize-button").click();

    cy.wait("@datasetQuery").then(({ response }) => {
      expect(response.statusCode).to.eq(200);
      // 验证返回的数据行结构
      const rows = response.body.data.rows;
      expect(rows.length).to.be.greaterThan(0);
      expect(rows[0]).to.have.length(2); // Category 与 Sum(Total) 两列
    });

    // 5. 验证前端数据表格已正确渲染各品类标签
    cy.get(".Table-cell").should("contain", "Doohickey");
    cy.get(".Table-cell").should("contain", "Gadget");
  });
});
```

---

### 3.2 功能二：行级数据沙箱（Data Sandboxing）越权防护断言

#### 3.2.1 业务功能背景
企业租户中，华北区销售经理只能看到 `Region = 'North'` 的数据行，即便该用户直接修改前端请求中的 Payload，后端也必须强制注入行级过滤条件。

#### 3.2.2 测试用例设计方案
* **测试用例实现（Clojure 后端驱动与权限测试）**：
```clojure
(deftest row-level-sandboxing-enforcement-test
  (testing "Sandboxed user query automatically appends security constraints"
    ;; 1. 构造普通销售员会话（被沙箱限制仅能访问 Region='North'）
    (mt/with-test-user :sales_rep_north
      (let [query (mt/mbql-query orders
                    {:fields [$id $total $region]})
            ;; 2. 模拟发起查询执行
            result (qp/process-query query)]
        
        ;; 3. 断言底层实际生成的 SQL 包含强制性的 WHERE 过滤
        (is (str/includes? (-> result :data :native_form :query)
                           "\"orders\".\"region\" = 'North'"))
        
        ;; 4. 断言返回的结果集中绝不包含其它大区的数据
        (let [regions (set (map #(nth % 2) (-> result :data :rows)))]
          (is (= #{"North"} regions)))))))
```

---

### 3.3 功能三：前端慢查询取消与数据库端连接回收

#### 3.3.1 业务功能背景
当用户发起一个耗时 30 秒的复杂全表扫描查询后，在页面点击“Cancel”或关闭当前浏览器选项卡。系统必须向目标数据库下发 `pg_cancel_backend(pid)`，释放该查询占用的连接与内存。

#### 3.3.2 测试用例设计方案
* **测试实现步骤**：
  1. 通过测试夹具向后端提交一个故意包含 `pg_sleep(10)` 的查询。
  2. 获取该查询在数据库端的执行进程 PID。
  3. 前端或测试客户端向 `/api/dataset/:query_id/cancel` 发送取消指令。
  4. 轮询目标数据库的系统视图 `pg_stat_activity`。
  5. **断言**：该进程 PID 在 500ms 内状态转为 IDLE，没有长期存活的慢查询残留。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Dockerized Multi-Database Test Harness**：
   * CI 流水线启动专有 Docker 容器组，预置标准的 `Sample Database`（包含 Orders, People, Products, Reviews 标准数据集），供所有方言驱动执行基准一致性对比。
2. **Cypress 测试分片（16 并发 CI Runner）**：
   * 将数百个大盘与交互用例按文件标签自动均分至 16 台云端测试机，使端到端全量回归耗时由 90 分钟压减至 8 分钟。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **将 AST 查询树作为测试的一等公民**：对于报表、低代码、数据平台，前端 UI 操作产生的是结构化 AST，后端转译的也是 AST；针对 AST 节点进行单向/双向转译的纯函数测试，比直接跑端到端能快几十倍。
* **安全沙箱测试必须同时断言“生成的 SQL”与“物理结果集”**：权限测试不仅要断言数据未泄露，还要检查生成的物理 SQL 是否被正确拼接了参数化绑定，防止 SQL 注入。
* **资源取消（Cancellation）链路的必须验证**：任何支持大耗时数据计算的系统，必须在自动化用例中测试“主动中断”链路的闭环，这是保证高可用系统连接池不被耗尽的生命线。
