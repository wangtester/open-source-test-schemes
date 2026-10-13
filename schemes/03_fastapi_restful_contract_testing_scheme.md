# FastAPI 自动化测试方案与契约校验全集深度剖析

## 1. 项目背景与系统架构全景

[FastAPI](https://github.com/fastapi/fastapi) 是当今 Python 生态中性能最高、采用最广泛的现代化 Web 框架。其核心依托 Starlette 处理高性能 ASGI 异步网络通信，依托 Pydantic V2 实现极速的 C 扩展级数据契约校验与序列化，并内建基于依赖注入（Dependency Injection）的架构模式与 OpenAPI / Swagger 规范自动生成。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        HTTP / WebSocket Client                         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ ASGI (Uvicorn / AnyIO)
┌───────────────────────────────────▼────────────────────────────────────┐
│                    FastAPI Core Framework Layer                        │
├───────────────────────────────┬────────────────────────────────────────┤
│ Dependency Injection Engine   │ Pydantic V2 Contract & Validation      │
│ - 递归依赖解析树 (Depends)    │ - Request Body / Query 参数严格类型断言│
│ - yield 上下文生命周期与资源清理│ - Response Model 自动过滤敏感字段       │
├───────────────────────────────┼────────────────────────────────────────┤
│ OpenAPI & JSON Schema Engine  │ Security & Authentication Subsystem    │
│ - 路由反射与交互式文档生成    │ - OAuth2 with Password (and hashing)   │
│ - 状态码与多媒体类型规范映射   │ - Scopes 细粒度权限校验与 JWT 验证      │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

FastAPI 自身的测试套件兼备了微秒级契约单元断言与高吞吐异步 ASGI 全链路回归：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Pydantic Validation Unit** | 畸形参数拦截、422 Unprocessable Entity 格式校验、类型自动转换 | Pytest + Pydantic | 纯数据结构输入输出比对 |
| **Dependency Override Tests** | 使用 `app.dependency_overrides` 覆盖数据库连接、认证 Token | TestClient (Starlette) | 内存字典重写，用例结束后必须 `.clear()` |
| **Response Model Masking** | 响应模型中排除 `password_hash`、敏感内部字段屏蔽验证 | Pytest + TestClient | 响应体字段全等对比 |
| **OpenAPI Schema Tests** | 生成的 `/openapi.json` 规范与 JSON Schema 3.1 兼容性 | Pytest + openapi-spec-validator | Golden JSON 规范快照比对 |
| **Async / Streaming Tests** | SSE 流式生成器提前断开、WebSocket 全双工回显、取消信号传播 | `httpx.AsyncClient` + AnyIO | 异步事件循环与本地 ASGI 直连 |
| **Security & OAuth2 Scopes** | JWT 签名过期、非法算法篡改、Scope 权限不足返回 403 | Pytest + PyJWT / Passlib | 构造包含不同 Payload 的伪造 Token |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：Pydantic V2 数据契约校验与 422 错误定位测试

#### 3.1.1 业务功能与测试目标
当客户端传入非法类型（如期望数字却传入字符串，或者缺失必填属性）时，FastAPI 必须返回标准 422 响应，并在 `detail` 中精准指明出错字段位置（`loc`）与错误消息（`msg`）。

#### 3.1.2 测试用例实现
```python
import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field, EmailStr

app = FastAPI()

class UserRegisterRequest(BaseModel):
    username: str = Field(min_length=3, max_length=20)
    email: EmailStr
    age: int = Field(gt=0, le=120)

@app.post("/users/register")
def register_user(req: UserRegisterRequest):
    return {"status": "ok", "user": req.username}

client = TestClient(app)

def test_pydantic_validation_error_structure():
    # 1. 传入缺失 email 且 age 为负数的畸形数据
    invalid_payload = {
        "username": "ab", # 长度不足 3
        "age": -5
    }

    response = client.post("/users/register", json=invalid_payload)

    # 2. 状态码与错误结构精准断言
    assert response.status_code == 422
    data = response.json()
    assert "detail" in data

    errors = data["detail"]
    assert len(errors) == 3

    # 校验 username 错误
    user_err = next(e for e in errors if e["loc"] == ["body", "username"])
    assert "string_too_short" in user_err["type"]

    # 校验 email 缺失错误
    email_err = next(e for e in errors if e["loc"] == ["body", "email"])
    assert email_err["type"] == "missing"

    # 校验 age 范围错误
    age_err = next(e for e in errors if e["loc"] == ["body", "age"])
    assert "greater_than" in age_err["type"]
```

---

### 3.2 模块二：依赖注入覆盖（Dependency Overrides）与事务生命周期测试

#### 3.2.1 业务功能与测试目标
在集成测试中，生产环境的数据库会话（`get_db`）必须被测试专用的内存数据库会话覆盖，且 `yield` 之后的资源清理逻辑（如关闭连接或事务回滚）必须被执行。

#### 3.2.2 测试用例实现
```python
import pytest
from fastapi import FastAPI, Depends
from fastapi.testclient import TestClient

app = FastAPI()

# 生产环境依赖
def get_db():
    raise RuntimeError("Cannot connect to production database during test!")

@app.get("/items")
def list_items(db=Depends(get_db)):
    return db.query_items()

def test_dependency_override_and_teardown():
    cleanup_called = False

    # 测试环境专用依赖
    def override_get_db():
        nonlocal cleanup_called
        class FakeSession:
            def query_items(self):
                return ["item1", "item2"]
        try:
            yield FakeSession()
        finally:
            cleanup_called = True # 模拟连接关闭

    # 1. 注入覆盖
    app.dependency_overrides[get_db] = override_get_db
    client = TestClient(app)

    try:
        response = client.get("/items")
        assert response.status_code == 200
        assert response.json() == ["item1", "item2"]
        # 2. 断言 yield 之后的清理逻辑被正常触发
        assert cleanup_called is True
    finally:
        # 3. 必须清空覆盖，杜绝测试污染
        app.dependency_overrides.clear()
```

---

### 3.3 模块三：Response Model 敏感字段物理屏蔽测试

#### 3.3.1 业务功能与测试目标
后端数据库 ORM 对象通常包含 `hashed_password` 与内部密钥。FastAPI 必须根据路由声明的 `response_model` 严格剔除未声明的敏感属性。

#### 3.3.2 测试用例实现
```python
from fastapi import FastAPI
from fastapi.testclient import TestClient
from pydantic import BaseModel

app = FastAPI()

class UserInDB:
    def __init__(self, username, hashed_pwd, internal_secret):
        self.username = username
        self.hashed_password = hashed_pwd
        self.internal_secret = internal_secret

class UserPublicResponse(BaseModel):
    username: str

@app.get("/user/me", response_model=UserPublicResponse)
def get_current_user():
    return UserInDB("admin", "pbkdf2_sha256$hashed", "SUPER_SECRET_KEY")

client = TestClient(app)

def test_response_model_masks_sensitive_data():
    response = client.get("/user/me")
    assert response.status_code == 200
    json_data = response.json()

    # 1. 验证允许的公开字段存在
    assert json_data["username"] == "admin"

    # 2. 强断言：敏感字段被完全剔除
    assert "hashed_password" not in json_data
    assert "internal_secret" not in json_data
```

---

### 3.4 模块四：OpenAPI 3.1 规范模式（JSON Schema）自动化校验测试

#### 3.4.1 业务功能与测试目标
测试必须验证系统动态生成的 `/openapi.json` 能够被标准 OpenAPI 校验器成功解析，所有的请求参数、返回类型、安全 Scopes 定义均无 Schema 破损。

#### 3.4.2 测试用例实现
```python
import pytest
from fastapi.testclient import TestClient

def test_openapi_schema_generation_and_validity():
    from fastapi import FastAPI
    app = FastAPI(title="Sample Service", version="1.0.0")

    @app.get("/ping")
    def ping():
        return {"ping": "pong"}

    client = TestClient(app)
    response = client.get("/openapi.json")

    assert response.status_code == 200
    schema = response.json()

    # 结构完整性断言
    assert schema["openapi"].startswith("3.1.")
    assert schema["info"]["title"] == "Sample Service"
    assert "/ping" in schema["paths"]
    assert "get" in schema["paths"]["/ping"]
```

---

## 4. 测试工程化与基础设施

1. **AnyIO 跨异步引擎矩阵**：
   * FastAPI 使用 AnyIO 抹平 Asyncio 与 Trio 的差异，在 CI 中矩阵运行两大事件循环，确保底层异步兼容性。
2. **零启动耗时 TestClient**：
   * 基于 Starlette 的内存通道测试客户端，直接调用 ASGI 应用接口，无需在测试机上实际绑定网络端口，单个测试用例耗时低于 1 毫秒。

---

## 5. 对同类复杂系统的参考价值

* **声明式依赖注入与测试隔离**：将数据库、配置、鉴权完全设计为可通过字典覆盖的依赖项（Dependency Overrides），是开发高可测、无外部依赖纯净单元测试的黄金范式。
