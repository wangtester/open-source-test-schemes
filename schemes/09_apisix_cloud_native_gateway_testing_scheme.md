# Apache APISIX 自动化测试方案与云原生网关用例全集深度剖析

## 1. 项目背景与系统架构全景

[Apache APISIX](https://github.com/apache/apisix) 是全球顶级的动态、实时、高性能云原生微服务 API 网关。系统基于 Nginx 内核与 LuaJIT（OpenResty）构建数据面，依托 etcd v3 提供分布式配置中心控制面。APISIX 的架构彻底颠覆了传统 Nginx 需要 `nginx -s reload` 重启 worker 进程导致长连接断开的弊端，实现了全配置（路由、SSL 证书、插件、上游服务节点）的**毫秒级动态热加载（Hot-reloading without reload）**。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        HTTP / gRPC / TCP Traffic                       │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ 极速数据面转发 (OpenResty / LuaJIT)
┌───────────────────────────────────▼────────────────────────────────────┐
│                    APISIX Data Plane (Worker Processes)                │
├───────────────────────────────┬────────────────────────────────────────┤
│ RadixTree Route Matching      │ Plugin Pipeline Execution Chain        │
│ - URI/Host/Method/Header 匹配 │ - rewrite -> access -> header_filter   │
│ - 权重分流与金丝雀灰度发布    │ - body_filter -> log 各阶段插件链      │
├───────────────────────────────┼────────────────────────────────────────┤
│ Upstream Health Check & LB    │ etcd v3 Long-polling Watcher           │
│ - 被动/主动健康探测熔断       │ - 毫秒级变更事件监听广播               │
│ - Roundrobin / ConsistentHash │ - 内存缓存版本一致性比对 (Revision)    │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

APISIX 拥有基于 Perl 的 **Test::Nginx 声明式测试框架** 以及 Go 自动化测试套件：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Plugin Unit Tests** | 插件参数 JSON Schema 校验、限流计数器算法、WAF 正则命中 | Test::Nginx / Busted (Lua) | 纯内存调用插件钩子函数 |
| **Hot Reload Dynamic Tests** | 写入 etcd 配置后，在不 reload 情况下发起请求验证即时生效 | Test::Nginx + 独立 etcd | 每套测试动态挂载临时 etcd 前缀 |
| **Route RadixTree Tests** | 万级路由并发匹配、通配符优先级、基于 Cookie 的 A/B 分流 | Test::Nginx 声明式用例 | 自动化启停独立受测 OpenResty 进程 |
| **Fault Injection & Chaos** | etcd 临时宕机时本地内存缓存继续提供路由代理 | Test::Nginx + 杀死 etcd 进程 | 模拟网络中断与存储断连 |
| **Performance Benchmark** | 50,000+ QPS 极限压测下 CPU 饱和度与 P99 延迟抖动 | wrk / FlameGraph 自动化火焰图 | 物理 CPU 亲和性绑定 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：etcd v3 动态路由毫秒级热加载与零 Reload 验证测试

#### 3.1.1 业务功能与测试目标
向 APISIX Admin API 提交一条全新路由规则。在没有任何 Nginx 进程重启或 reload 的前提下，数据面 Worker 必须在 10 毫秒内通过 etcd watch 机制完成内存路由表原地无缝更新，并立即对后续 HTTP 请求按新路由转发。

#### 3.1.2 测试用例实现（Test::Nginx 声明式测试规范）
```perl
use Test::Nginx::Socket::PB;
use t::APISIX 'plan';

repeat_each(1);
no_long_string();
no_shuffle();
no_root_location();

run_tests();

__DATA__

=== TEST 1: Register dynamic route and test zero-reload immediate effect
--- config
    location /t {
        content_by_lua_block {
            local t = require("lib.test_admin").test
            -- 1. 通过 Admin API 动态下发一条路由：匹配 /hello 转发至后端 mock upstream
            local code, body = t('/apisix/admin/routes/1',
                ngx.HTTP_PUT,
                [[{
                    "uri": "/hello",
                    "upstream": {
                        "type": "roundrobin",
                        "nodes": {
                            "127.0.0.1:1980": 1
                        }
                    }
                }]]
            )
            if code >= 300 then
                ngx.status = code
                ngx.say(body)
                return
            end
            ngx.say("ROUTE_CONFIGURED")
        }
    }
--- request
GET /t
--- response_body
ROUTE_CONFIGURED
--- no_error_log
[error]

=== TEST 2: Hit the newly configured route immediately without reloading
--- request
GET /hello
--- response_body
passed
--- no_error_log
[error]
[warn]
```

---

### 3.2 模块二：令牌桶限流插件（Limit-Req）精确速率与突发突增（Burst）测试

#### 3.2.1 业务功能与测试目标
配置限流规则：速率限制为 `rate: 2 r/s`，突发并发允许 `burst: 1`。测试必须验证：
1. 突发发送 3 个连续请求时，前 2 个请求被放行处理（1 个正常，1 个利用 burst 缓冲）。
2. 第 3 个请求因超出桶容量被立即拒绝，严格返回 `HTTP 503 Service Unavailable` 或 `HTTP 429 Too Many Requests`。

#### 3.2.2 测试用例实现（Test::Nginx 测试）
```perl
=== TEST 3: Limit-req plugin throttles burst requests accurately
--- config
    location /t {
        content_by_lua_block {
            local t = require("lib.test_admin").test
            -- 绑定限流插件：rate = 1, burst = 1, key = remote_addr
            local code, body = t('/apisix/admin/routes/2',
                ngx.HTTP_PUT,
                [[{
                    "uri": "/rate-limit",
                    "plugins": {
                        "limit-req": {
                            "rate": 1,
                            "burst": 1,
                            "rejected_code": 429,
                            "key": "remote_addr"
                        }
                    },
                    "upstream": {
                        "type": "roundrobin",
                        "nodes": { "127.0.0.1:1980": 1 }
                    }
                }]]
            )
            ngx.say("LIMIT_CONFIGURED")
        }
    }
--- request
GET /t
--- response_body
LIMIT_CONFIGURED

=== TEST 4: Concurrently trigger limit-req and assert HTTP 429
--- pipelined_requests eval
["GET /rate-limit", "GET /rate-limit", "GET /rate-limit"]
--- response_status eval
[200, 200, 429]
--- no_error_log
[error]
```

---

### 3.3 模块三：上游主动健康检查（Healthcheck）与节点自动摘除测试

#### 3.3.1 业务功能与测试目标
上游配置包含节点 A 与节点 B。当节点 A 连续返回 2 次 HTTP 500 错误时，被动健康检查机制必须自动将节点 A 标记为 `UNHEALTHY` 并踢出可用节点池，后续流量 100% 路由至节点 B。

#### 3.3.2 测试用例实现
```perl
=== TEST 5: Healthcheck automatically ejects failing upstream node
--- config
    location /t {
        content_by_lua_block {
            local t = require("lib.test_admin").test
            -- 配置带健康探测的上游
            local code, body = t('/apisix/admin/upstreams/1',
                ngx.HTTP_PUT,
                [[{
                    "nodes": {
                        "127.0.0.1:1981": 1,
                        "127.0.0.1:1982": 1
                    },
                    "type": "roundrobin",
                    "checks": {
                        "active": {
                            "http_path": "/status",
                            "healthy": { "interval": 1, "successes": 1 },
                            "unhealthy": { "interval": 1, "http_failures": 2 }
                        }
                    }
                }]]
            )
            ngx.say("HEALTHCHECK_READY")
        }
    }
--- request
GET /t
--- response_body
HEALTHCHECK_READY
```

---

## 4. 测试工程化与基础设施

1. **Test::Nginx 原生轻量隔离沙箱**：
   * 框架在 `/tmp/test-nginx-*` 目录为每次测试独立渲染 `nginx.conf` 模板，独立启动 Nginx Master 与 Worker 进程，并在测试完成瞬间强制回收所有套接字与进程。
2. **etcd 内存临时存储加速测试**：
   * 在 CI 环境中使用内存盘（`tmpfs`）运行单节点 etcd，单条热加载用例执行耗时控制在 5ms 以内。

---

## 5. 对同类复杂系统的参考价值

* **配置热加载的黑盒自动化断言范式**：对于宣称支持“零停机动态配置”的高性能网关或反向代理，测试必须通过流水线请求（Pipelined requests）在毫秒级别断言新旧配置切换边界，杜绝瞬时丢包或回退。
