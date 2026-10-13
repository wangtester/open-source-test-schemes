# OpenWebUI 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[OpenWebUI](https://github.com/open-webui/open-webui)（原 Ollama WebUI）是全球用户体量最大、功能最全面的开源 AI Web 交互平台与企业级私有化大模型网关。系统后端基于 Python (FastAPI) + SQLAlchemy 构建，前端采用 SvelteKit + Tailwind CSS 构建现代化全功能交互控制台，内置企业级多租户 RBAC、内置知识库（Documents RAG）、语音交互、Modelfile 镜像管理与多模型聚合路由。

### 1.2 被测系统核心组成
1. **多租户角色权限访问控制（RBAC & User Management）**：管理员、普通用户、受限用户的多级权限隔离，控制模型访问列表与全局知识库。
2. **Modelfile 视觉化定制引擎**：支持在 UI 界面声明式定义 System Prompt、Temperature、上下文窗口与参数并生成虚拟衍生模型。
3. **文档知识库与检索管线（Knowledge & Documents RAG）**：文档解析切片、向量嵌入、会话内引用角标（Citations）与源文本弹窗。
4. **多模型分屏对比（Split-screen Arena）**：在同一页面驱动多个模型就同一问题并发输出打字机流。

### 1.3 自动化测试的主要难点与挑战
* **RBAC 权限边界的防越权测试覆盖度**：普通用户直接调用管理员 API、查看他人私密对话或未授权模型的边界拦截。
* **分屏多模型并发打字机渲染的性能与同步**：同时渲染 3 个模型的 SSE 流时前端 DOM 节点刷新压力。
* **文件上传切片与知识库持久化的端到端闭环**：验证从网页拖入 PDF 到会话中引用角标点亮的完整链路。

---

## 2. 测试架构全景与分层策略

OpenWebUI 建立了覆盖多角色权限与全栈交互的自动化测试防线：

```
┌────────────────────────────────────────────────────────┐
│             Cypress / Playwright Full-Stack E2E        │  用户交互层：多角色登录、Modelfile 配置、分屏对话体验
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         FastAPI Role-Based Access Control API Tests    │  权限中间件层：验证未授权 Token 访问受管路由的 403 阻断
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Knowledge Ingestion & Vector Retrieval Pipeline   │  知识库层：单测验证文档分块、检索引用角标生成
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         SQLAlchemy DB Migrations & User Schema Tests   │  持久化底座：验证数据库在 SQLite 与 Postgres 下的数据迁移
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：RBAC 多角色越权阻断测试（Security Gating）

#### 3.1.1 业务功能背景
管理员在后台将模型 `gpt-4o-enterprise` 的可见范围设置为仅允许 `Admin`。普通用户 `user_bob` 登录后，前端模型选择器不得出现该模型；且若 `user_bob` 绕过前端直接调用 POST 请求尝试使用该模型发起会话，后端必须返回 403 Forbidden。

#### 3.1.2 测试用例设计方案
* **测试用例实现（FastAPI TestClient 集成测试）**：
```python
import pytest
from fastapi.testclient import TestClient

def test_regular_user_cannot_access_restricted_admin_model(client, admin_token, user_token):
    # 1. 管理员创建受限制的模型实体
    client.post(
        "/api/v1/models/create",
        headers={"Authorization": f"Bearer {admin_token}"},
        json={"id": "admin-only-model", "name": "Admin Only", "access_control": {"read": ["admin"]}}
    )

    # 2. 普通用户获取可用模型列表
    resp_list = client.get("/api/v1/models", headers={"Authorization": f"Bearer {user_token}"})
    assert resp_list.status_code == 200
    model_ids = [m["id"] for m in resp_list.json()["data"]]
    # 视图层硬断言：列表中绝对不出现受限制模型
    assert "admin-only-model" not in model_ids

    # 3. 普通用户尝试直接调用该模型发起对话
    resp_chat = client.post(
        "/api/v1/chat/completions",
        headers={"Authorization": f"Bearer {user_token}"},
        json={"model": "admin-only-model", "messages": [{"role": "user", "content": "hi"}]}
    )
    # 数据面硬断言：直接物理拦截返回 403
    assert resp_chat.status_code == 403
    assert "not authorized" in resp_chat.json()["detail"].lower()
```

---

### 3.2 功能二：知识库引用角标（Citations）与源文本锚点跳转

#### 3.2.1 业务功能背景
用户上传一份政策文档并绑定到当前对话。提问后模型在回答中输出了包含 `[1]` 的角标。点击该角标，右侧必须弹出引用文档详情，且对应段落被黄色高亮渲染。

#### 3.2.2 测试用例设计方案
* **测试用例实现（Playwright E2E 交互测试脚本）**：
```typescript
test('clicking citation badge displays source document modal with highlighted text', async ({ page }) => {
  await page.goto('/c/test-chat-with-rag');

  // 1. 定位会话回答中的第一个引用角标 [1]
  const citationBadge = page.locator('.citation-badge').first();
  await expect(citationBadge).toBeVisible();

  // 2. 点击角标激活源文本详情弹窗
  await citationBadge.click();

  // 3. 断言弹窗呈现源文档标题与高亮段落
  const citationModal = page.locator('#citation-detail-modal');
  await expect(citationModal).toBeVisible();
  await expect(citationModal.locator('.source-filename')).toHaveText('company_policy.pdf');
  await expect(citationModal.locator('.highlighted-chunk')).toHaveCSS('background-color', 'rgb(254, 240, 138)'); // 验证黄色高亮 CSS
});
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Dockerized 容器镜像发布烟雾测试（Post-Build Smoke）**：
   * 在 GitHub Actions 打包出正式 Docker 镜像后，通过 `docker run -d -p 8080:8080` 启动容器，向 `/health` 发送探测并执行真实注册登录流程，确保最终发布镜像零缺陷。
2. **多数据库后端迁移测试**：
   * 在每次提交时针对 SQLite 与 PostgreSQL 分别跑一遍 Alembic 数据库迁移脚本，断言元数据表平滑升级与回滚。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **将“多角色越权审计”列为最高安全防线**：在企业级 AI 门户系统中，防越权不仅要在前端做隐藏，更要在数据面 API 执行物理阻断；自动化测试必须设计覆盖多角色的正反向双轨用例。
* **交互式引用（Citations）必须断言“高亮与弹窗闭环”**：对于带来源依据的知识库系统，测试不能只看文字，必须验证点击角标后前端能否精准定位并高亮源文档段落。
