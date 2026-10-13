# Ollama 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Ollama](https://github.com/ollama/ollama) 是全球最普及的本地大模型运行、管理与打包部署框架。系统架构采用 Go 语言作为常驻后台服务与 RESTful API 管理网关，底层通过 C/C++ 动态链接绑定 `llama.cpp` 原生推理引擎，深度适配 Apple Metal、NVIDIA CUDA、AMD ROCm 以及 Intel CPU 加速。

### 1.2 被测系统核心组成
1. **模型分发与 Modelfile 引擎**：类似 Docker 镜像规范，支持 `FROM`, `PARAMETER`, `TEMPLATE`, `SYSTEM` 声明式定义并生成不可变模型清单（Manifest & Blobs）。
2. **GGUF 模型文件解析器与量化加载器**：解析 GGUF 格式头部、层级张量、量化类型（Q4_K, Q8_0）与上下文长度。
3. **并发调度与生命周期管理（Keep-Alive & VRAM Eviction）**：并发请求队列排队、显存空闲自动卸载机制（Keep-Alive Timeout）以释放硬件资源。
4. **OpenAI 兼容与原生流式 API**：提供基于 HTTP JSON 与 SSE 的流式文本/工具调用接口。

### 1.3 自动化测试的主要难点与挑战
* **C/Go 跨语言 CGO 调用的指针与内存安全**：llama.cpp 崩溃可能直接导致整个 Go 进程段错误（Segmentation Fault）。
* **跨平台异构硬件驱动的适配测试**：macOS (Metal)、Windows (MSVC/DirectML) 与 Linux (CUDA) 的动态链接库加载。
* **模型文件体积庞大（数 GB）**：自动化测试不可能在每次 CI 运行中下载 7B/14B 大模型。

---

## 2. 测试架构全景与分层策略

Ollama 建立了使用微型模型（Toy Models）与桩引擎结合的轻量化自动化测试架构：

```
┌────────────────────────────────────────────────────────┐
│           Cross-Platform CLI & Native Binary E2E       │  发布产物层：在 macOS/Windows/Linux 验证打包安装包运行
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Go HTTP Server & Streaming API Tests           │  网关接口层：测试并发会话、流式分块、Keep-Alive 超时卸载
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│      Modelfile Parser & Layer Blob Digest Tests        │  打包镜像层：纯 Go 验证类似 Docker 层的哈希合并与清单
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│       Toy GGUF Model (<5MB) Real-Inference Smoke       │  推理真实层：基于极小测试模型执行实际 Token 生成断言
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：超轻量 Toy GGUF 模型端到端真实推理测试

#### 3.1.1 业务功能背景
如何证明整个 CGO 桥接、显存分配、Prompt 模板格式化与自回归生成代码链路绝对可用，而无需占用数百兆网络带宽与显存？

#### 3.1.2 测试用例设计方案
Ollama 团队专门烘焙了一个仅有几兆大小的受测模型 `toy-model.gguf`（层数极浅、词表极小），直接签入代码仓库或内建生成。
* **测试用例实现（Go 语言测试用例）**：
```go
func TestGenerateWithToyModel(t *testing.T) {
    // 1. 初始化测试服务器，加载仓库内置的微型测试模型
    srv := setupTestServer(t)
    defer srv.Close()

    req := api.GenerateRequest{
        Model:  "toy-model",
        Prompt: "hello",
        Stream: false,
    }

    // 2. 发起生成调用
    var resp api.GenerateResponse
    err := srv.Client().Generate(context.Background(), &req, func(r api.GenerateResponse) error {
        resp = r
        return nil
    })

    // 3. 严格断言
    assert.NilError(t, err)
    assert.Assert(t, len(resp.Response) > 0, "Response tokens should not be empty")
    assert.Assert(t, resp.Done, "Response should be marked done")
    assert.Assert(t, resp.TotalDuration > 0, "Total duration should be recorded")
}
```

---

### 3.2 功能二：显存 Keep-Alive 自动卸载生命周期状态机

#### 3.2.1 业务功能背景
用户配置 `keep_alive = "1s"`。模型完成响应后，系统在 1 秒内应保持模型常驻内存（下次请求零冷启动）；当超过 1 秒无新请求时，后台守护协程必须自动调用底层 C++ 释放模型权重与上下文，让 GPU 显存占用归零。

#### 3.2.2 测试用例设计方案
* **测试编排**：
  1. 发送推理请求，设置 `keep_alive = "500ms"`。
  2. 请求完成后立即检查系统活动模型列表 `server.loadedModels()`，断言模型处于 `ACTIVE` 状态。
  3. 推进物理时间等待 600ms。
  4. 再次检查 `server.loadedModels()`。
  5. **断言**：模型已被后台自动垃圾回收并从显存卸载，列表为空。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **跨平台 Matrix 无头 Runner 编译**：
   * 在 GitHub Actions 中并行矩阵编译：`windows-latest`, `macos-latest`, `ubuntu-latest`，在构建产物阶段即时执行 `go test ./...`。
2. **断点续传与 Digest 校验测试**：
   * 针对大文件拉取，测试客户端通过 HTTP Range Header 模拟在 50% 处突然断网，断言重新发起请求时能从断点处继续拉取并最终通过 SHA256 校验。

---

## 5. 对复杂 AI 系统测试设计的实战借鉴意义

* **善用“极小模型（Toy Models）”做物理通路烟雾测试**：不需要动辄加载几十 G 的生产大模型，用几兆的特制微型模型跑通张量加载和计算流程，能大幅降低 CI 成本并保证真实硬件通道被覆盖。
* **资源持有型系统必须严格断言“闲置卸载（Eviction）”**：任何占用显存、长连接等贵重资源的常驻系统，自动化测试必须覆盖带有时间维度的超时释放状态机。
