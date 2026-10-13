# Apache APISIX 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Apache APISIX](https://github.com/apache/apisix) 是顶级 Apache 基金会开源的高性能云原生 API 网关与微服务流量管理平台。系统底层基于 OpenResty (Nginx + LuaJIT) 高性能网络库，控制面深度依赖 etcd 进行配置分发与动态监听，支持全动态路由、证书热加载与强大的插件链机制。

### 1.2 被测系统核心组成
1. **控制面与 Admin API**：通过 RESTful 接口管理路由、上游、消费者、SSL 凭据，并实时同步至 etcd。
2. **数据面引擎（Data Plane Engine）**：高性能反向代理流管道，实现毫秒级动态路由匹配与负载均衡。
3. **插件运行时（Plugin Pipeline）**：在 Nginx 七大生命周期阶段（`rewrite`, `access`, `before_proxy`, `header_filter`, `body_filter`, `log`）动态执行插件逻辑（限流、鉴权、日志、请求重写）。
4. **健康检查与熔断器（Healthcheck & Circuit Breaker）**：主动/被动探测上游节点健康度，自动剔除故障节点。

### 1.3 自动化测试的主要难点与挑战
* **C / Nginx / Lua 跨语言混合体测试复杂度高**：传统的单元测试框架难以有效驱动 Nginx 多 Worker 进程与底层共享内存（`lua_shared_dict`）。
* **动态配置秒级生效与零停机断言**：在不执行 `nginx -s reload` 的前提下，如何验证新路由是否真正零停机、毫秒级热生效。
* **高并发网络请求与报文篡改**：网关涉及对 HTTP Header、Body、WebSocket、gRPC 协议的双向拦截与修改，边缘场景极多。

---

## 2. 测试架构全景与分层策略

Apache APISIX 采用了业界著名的 **`Test::Nginx` 声明式测试框架**，结合真实的 etcd 集群构建端到端回归防线：

```
┌────────────────────────────────────────────────────────┐
│             Test::Nginx Declarative E2E Suite          │  声明式网络层：模拟真实客户端 HTTP 报文收发
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Dynamic Admin API & etcd Sync Tests               │  控制面同步层：Admin API 注入规则，断言数据面生效时差
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│       Plugin Lifecycle Phase Pipeline Tests            │  插件管道层：分阶段断言 rewrite / access / body 拦截
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Mock Upstream & Fault Injection                │  上游桩服务：搭建延迟、500 报错、断开连接的 Fake Upstream
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：基于 `Test::Nginx` 的声明式 HTTP 报文测试

#### 3.1.1 业务功能背景
网关测试的本质是：输入特定的 HTTP 请求，断言输出的 HTTP 响应（状态码、Header、Body、错误日志）。命令式测试代码往往冗长，APISIX 采用声明式格式。

#### 3.1.2 测试用例设计方案
* 每个 `.t` 测试文件由若干个声明式区块组成，测试运行器会自动启动定制化的 Nginx 进程执行驱动。
* **典型声明式用例示例**：
```perl
=== TEST 1: Hit dynamic route with proxy-rewrite plugin
--- config
    location /t {
        content_by_lua_block {
            local http = require("resty.http")
            local httpc = http.new()
            
            -- 1. 通过 Admin API 注册一条带重写插件的路由
            local res, err = httpc:request_uri("http://127.0.0.1:9180/apisix/admin/routes/1", {
                method = "PUT",
                body = [[{
                    "uri": "/hello",
                    "plugins": {
                        "proxy-rewrite": {
                            "uri": "/world",
                            "headers": { "X-Test-Inject": "APISIX" }
                        }
                    },
                    "upstream": {
                        "type": "roundrobin",
                        "nodes": { "127.0.0.1:1980": 1 }
                    }
                }]]
            })
            ngx.say(res.status)
        }
    }
--- request
GET /t
--- response_body
200
--- no_error_log
[error]

=== TEST 2: Verify rewritten request arrives at upstream
--- request
GET /hello
--- response_body
passed to /world
--- response_headers
X-Test-Inject: APISIX
```

---

### 3.2 功能二：零停机动态路由热加载时延测试

#### 3.2.1 业务功能背景
传统 Nginx 修改配置需要 Reload，可能导致连接抖动或高并发瞬时丢失。APISIX 基于 etcd 监听机制实现实时热生效。

#### 3.2.2 测试用例设计方案
* **测试编排**：
  1. 并发客户端以 100 QPS 持续向 `/api/order` 发起请求，断言初始返回 404（路由不存在）。
  2. 自动化脚本调用 Admin API 创建 `/api/order`。
  3. 客户端持续探测，记录从 Admin API 返回成功到数据面首次返回 200 的时间差。
  4. **硬性断言**：生效时间差必须严格小于 1000ms（毫秒级同步），且在变更期间绝对不发生 TCP 丢包或 Connection Refused。

---

### 3.3 功能三：上游健康检查与自动熔断（Circuit Breaker）测试

#### 3.3.1 业务功能背景
当上游集群中有 1 台机器出现持续 502/503 报错时，网关应自动将其置为 Unhealthy，将后续流量全部路由到健康的备用机器。

#### 3.3.2 测试用例设计方案
* **测试步骤**：
  1. 使用内置 Mock 服务搭建两个上游端口：`Upstream-A (port 1980)`，`Upstream-B (port 1981)`。
  2. 配置网关上游为负载均衡，并开启 `healthcheck.active`。
  3. 通过测试脚本将 `Upstream-A` 切换为返回 500 状态码。
  4. 触发连续 3 次健康检查探测。
  5. 客户端再次发起业务请求：
     * **断言**：流量 100% 转发至 `Upstream-B`，`Upstream-A` 请求量归零。
     * **恢复断言**：将 `Upstream-A` 重新改为返回 200，经历连续成功探测后，断言流量自动恢复平分轮询。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **容器化依赖拓扑（etcd + Prometheus + Mock Upstreams）**：
   * 所有集成测试在 GitHub Actions 运行前，通过轻量级 Docker 容器挂载标准 etcd 实例，保证配置存储环境与生产环境一致。
2. **多操作系统与多架构覆盖**：
   * 在 Linux (x86_64, arm64) 及 macOS 上分别运行，确保 LuaJIT 与 C 模块的字节码与内存指针在不同架构下行为一致。

---

## 5. 对复杂系统自动化测试设计的实战借鉴意义

* **声明式报文测试是网络/中间件类系统的最优解**：对于涉及协议转换、代理分发、报文改写的网络系统，采用“请求块 + 预期响应块 + 错误日志黑名单”的声明式结构能让用例清晰度提升数倍。
* **零停机与热生效必须有确定性时延断言**：对于声称支持“动态加载”、“热更新”的系统，测试中必须在持续压测的背景下执行配置下发，验证生效时延与零抖动。
* **主动与被动健康检查必须搭配双向恢复测试**：测试故障熔断时，不仅要测试“故障发生时是否顺利摘除”，更要测试“故障自愈后是否平滑回切”，避免单向熔断导致的节点永久下线。
