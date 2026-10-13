# Moby (Docker) 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Moby](https://github.com/moby/moby) 是 Docker 开源的技术中枢，是云原生容器引擎事实上的工业标准。系统主要使用 Go 语言开发，深度调用 Linux 内核特性（Namespaces, Cgroups, OverlayFS, Seccomp, AppArmor）。

### 1.2 被测系统核心组成
1. **Docker Daemon (`dockerd`)**：常驻系统后台守护进程，管理镜像、容器、网络、存储卷及安全策略。
2. **Containerd & runc**：底层 OCI 标准容器运行时，负责真实的 Linux 进程生命周期隔离。
3. **Docker Engine API (Unix Socket / gRPC)**：通过 Unix 域套接字暴露的 RESTful/gRPC 控制接口。
4. **Docker CLI (`docker`)**：用户交互命令行工具，负责参数解析、格式化输出及与 Daemon 通信。

### 1.3 自动化测试的主要难点与挑战
* **对操作系统特权与内核特性的强依赖**：容器测试必须在具备 `root` 特权或支持真实内核命名空间的环境中运行。
* **守护进程的真实生命周期与残留治理**：测试异常中断时，极易残留未停止的容器、僵尸进程、挂载点泄漏或虚拟网卡残留。
* **异步守护进程通信与非结构化文本输出**：CLI 输出受终端大小、ANSI 转义序列及网络流式返回影响。

---

## 2. 测试架构全景与分层策略

Moby 项目演进出了极其完备的系统级自动化测试分层体系：

```
┌────────────────────────────────────────────────────────┐
│           Docker-in-Docker (DinD) E2E Test             │  全系统端到端：在隔离容器内启动全新 dockerd 与 CLI
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│         Integration Suite (daemon & API package)       │  集成层：通过 Go 测试夹具动态管理独立临时 Daemon
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         CLI Execution & icmd Output Assertions         │  命令测试层：模拟执行真实二进制并捕获 stdout/stderr
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│             Go Unit Tests (pkg/ & daemon/)             │  单元测试层：纯算法与逻辑测试（无特权需求）
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：独立测试守护进程（Ephemeral Test Daemon）生命周期

#### 3.1.1 业务功能背景
如果所有测试用例共享宿主机上同一个运行中的 `dockerd`，一个测试用例配置了错误的存储驱动或死锁将导致整个测试集崩溃。

#### 3.1.2 测试用例设计方案
Moby 封装了 `daemon.Daemon` 测试夹具：
* 每次测试或每个测试套件在独立的临时工作目录中冷启动一个全新的 `dockerd` 进程。
* 独立分配 Unix Socket 路径（如 `/tmp/test-dockerd-12345/docker.sock`）。
* 用例结束后执行标准清理：向 Daemon 发送 `SIGTERM`，等待进程退出；若超时则 `SIGKILL`，并自动卸载其挂载的 OverlayFS。
* **测试夹具示例（Go 语言）**：
```go
func TestDaemonCustomStorageDriver(t *testing.T) {
    // 1. 创建独立测试守护进程夹具
    d := daemon.New(t, daemon.WithRootDir(t.TempDir()))
    
    // 2. 传递特定启动参数并启动
    err := d.Start("--storage-driver=vfs", "--log-level=debug")
    assert.NilError(t, err)
    defer d.Stop(t) // 保证用例退出时关闭守护进程并卸载挂载点

    // 3. 通过该守护进程专有的 Client 发起操作断言
    info, err := d.NewClientT(t).Info(context.Background())
    assert.NilError(t, err)
    assert.Equal(t, info.Driver, "vfs")
}
```

---

### 3.2 功能二：基于 `icmd` 的命令行 CLI 端到端断言

#### 3.2.1 业务功能背景
Docker CLI 需要测试各种复杂的参数组合（Flag）、标准输入注入（`docker build - < Dockerfile`）、标准错误分流以及精确的进程退出码。

#### 3.2.2 测试用例设计方案
Moby 团队开源了 `gotest.tools/v3/icmd` 库，作为命令行自动化的专用断言套件。
* **测试用例实现**：
```go
func TestDockerRunExitCodeAndOutput(t *testing.T) {
    ctx := setupTest(t)

    // 1. 测试成功场景：断言退出码为 0，且 stdout 匹配
    result := icmd.RunCommand("docker", "run", "--rm", "busybox", "echo", "hello-world")
    result.Assert(t, icmd.Expected{
        ExitCode: 0,
        Out:      "hello-world\n",
        Err:      "",
    })

    // 2. 测试容器内命令失败：断言容器退出码透传
    failResult := icmd.RunCommand("docker", "run", "--rm", "busybox", "sh", "-c", "exit 42")
    failResult.Assert(t, icmd.Expected{
        ExitCode: 42,
    })

    // 3. 测试错误参数：断言 CLI 捕获并向 stderr 输出明确指引
    invalidResult := icmd.RunCommand("docker", "run", "--unknown-flag")
    invalidResult.Assert(t, icmd.Expected{
        ExitCode: 125, // Docker CLI 内部解析错误的特定退出码
        Err:      "unknown flag: --unknown-flag",
    })
}
```

---

### 3.3 功能三：容器资源隔离与 Cgroups 配额断言

#### 3.3.1 业务功能背景
当用户指定 `--memory 100m` 或 `--cpus 1.5` 时，如何测试容器是否真正受到底层 Linux Cgroups 的硬性限制，而不仅是参数被接收？

#### 3.3.2 测试用例设计方案
* **设计原则**：**“深入操作系统内核虚拟文件系统验证”**。
* **测试步骤**：
  1. 启动容器：`docker run -d --memory 104857600 (100MB) busybox sleep 100`。
  2. 测试程序通过 inspect 获取该容器在宿主机上的实际 `State.Pid`。
  3. 测试程序直接读取宿主机 `/sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes`（或 cgroup v2 对应的 `memory.max`）。
  4. 断言该内核控制文件的字节值精确等于 `104857600`。
  5. 验证超限熔断：注入内存压测程序，断言容器触发 OOMKilled 并被内核安全终止。

---

### 3.4 功能四：孤儿资源泄露自动检测防线

#### 3.4.1 业务功能背景
集成测试运行数小时后，测试节点可能会堆积成百上千个死容器或未删除的卷，导致后续测试因磁盘耗尽失败。

#### 3.4.2 测试用例设计方案
* 编写全局清理与泄漏巡检中间件：
```go
func TestMain(m *testing.M) {
    // 记录初始系统状态
    initialContainers := listAllContainers()
    
    // 执行所有测试
    code := m.Run()
    
    // 巡检清理
    remainingContainers := listAllContainers()
    if len(remainingContainers) > len(initialContainers) {
        log.Fatalf("Test suite leaked %d containers!", len(remainingContainers)-len(initialContainers))
    }
    os.Exit(code)
}
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **Docker-in-Docker (DinD) 容器化流水线**：
   * 所有集成测试本身运行在一个具备 `--privileged` 的专用容器内，确保与 CI 物理宿主机完全隔离，即使测试过程中执行了破坏性的系统调用（如修改路由表），也仅损坏该临时容器。
2. **多存储驱动矩阵（Overlay2, VFS, Btrfs）**：
   * 在不同的矩阵分支上分别启动支持不同文件系统后端驱动的守护进程，验证底层驱动兼容性。

---

## 5. 对复杂系统自动化测试设计的实战借鉴意义

* **复杂后台服务测试应采用“瞬时独立实例（Ephemeral Instance）”模式**：不要依赖一个长期运行的开发环境守护进程，而应通过自动化脚本在独立端口/路径即用即起、用完即毁。
* **使用专门的 CLI 运行断言工具库**：测试命令行工具时，退出码（Exit Code）、标准输出流（Stdout）与错误流（Stderr）必须分别捕获并独立断言。
* **操作系统内核级状态校验**：对于涉及系统级操作（磁盘配额、权限控制、网络防火墙）的软件，断言必须穿透到操作系统文件系统或内核状态层面。
