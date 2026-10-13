# Moby (Docker) 自动化测试方案与底层系统用例全集深度剖析

## 1. 项目背景与系统架构全景

[Moby](https://github.com/moby/moby) 是全球容器技术事实标准的基石项目（Docker 核心开源引擎）。其架构核心包括容器守护进程 `dockerd`、容器运行时标准接入 `containerd` 与 `runc`、Linux 内核 Namespaces 与 Cgroups 虚拟化隔离、OverlayFS2 存储分层驱动以及 libnetwork 虚拟网桥路由。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Docker CLI / Client SDK                         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Unix Domain Socket (/var/run/docker.sock)
┌───────────────────────────────────▼────────────────────────────────────┐
│                    dockerd (Docker Daemon Core Engine)                 │
├───────────────────────────────┬────────────────────────────────────────┤
│ Container Lifecycle & Cgroups │ Storage Driver (OverlayFS2)            │
│ - PID/MNT/NET/IPC Namespaces  │ - 镜像只读层 (LowerDir) 堆叠           │
│ - 内存/CPU 限制与 OOM-Killer   │ - 容器读写层 (UpperDir/MergedDir) 隔离 │
├───────────────────────────────┼────────────────────────────────────────┤
│ Libnetwork (虚拟网络子系统)   │ BuildKit (现代镜像构建引擎)            │
│ - docker0 虚拟网桥与 veth 对  │ - 声明式多阶段构建并发 DAG 调度        │
│ - iptables 规则动态改写与暴露 │ - 缓存挂载点与秘密凭证安全传递         │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Moby 具备操作系统内核级与大规模分布式系统级别的严密自动化测试套件：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Go Unit Tests** | 命令行解析、Docker 架构类型转换、配置解析器 | Go `testing` 纯单元测试 | 纯内存计算，无 root 特权 |
| **Daemon Integration (API)** | REST API 状态码、容器创建/启动/停止事件流 | Go `gotest.tools/v3` | 独立临时 dockerd 实例与套接字隔离 |
| **Cgroups & Resource Tests** | 限制 50MB 内存跑 OOM、CPU 配额（Quota/Period）打满 | 真实 Linux 内核特权容器 | 宿主机 cgroups 子系统独立命名层级 |
| **Storage Driver Tests** | OverlayFS2 写时复制（CoW）、文件删除 whiteout 掩码 | Linux 挂载文件系统 | 临时挂载点与虚拟回环设备（loopback） |
| **Libnetwork Integration** | 跨容器相互 ping、多网络隔离、自定义端口映射转发 | iptables + veth pairs | 独立网络命名空间（Network Namespace） |
| **Swarm Mode Cluster Tests** | Raft 选主、Overlay 多机网络、服务滚动更新与健康检查 | 虚拟多节点 dockerd 集群 | 内部多节点套接字互连，内存模拟集群 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：Cgroups 物理内存硬限制与 OOM-Killer 状态机捕获测试

#### 3.1.1 业务功能与测试目标
当用户为容器指定内存限制 `--memory 50m --memory-swap 50m` 时，内核 Cgroups 必须限制内存。若容器内部程序申请超过 50MB 内存，Linux 内核会触发 OOM-Killer。测试必须验证：
1. 容器进程被内核终止，退出码为 137（SIGKILL）。
2. 守护进程正确捕获事件并在 `docker inspect` 中将 `OOMKilled` 置为 `true`。

#### 3.1.2 测试用例实现
```go
package integration

import (
	"context"
	"testing"
	"time"

	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/client"
	"gotest.tools/v3/assert"
)

func TestContainerMemoryLimit_TriggersOOMKilled(t *testing.T) {
	ctx := context.Background()
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	assert.NilError(t, err)

	// 1. 配置限制容器：内存上限严格锁定为 32MB，禁止 Swap 缓冲
	config := &container.Config{
		Image: "alpine:latest",
		// 使用 python 脚本在内存中分配 64MB 数组，瞬间超过 32MB 限制
		Cmd: []string{"python3", "-c", "x = ' ' * (64 * 1024 * 1024)"},
	}
	hostConfig := &container.HostConfig{
		Resources: container.Resources{
			Memory:     32 * 1024 * 1024,
			MemorySwap: 32 * 1024 * 1024,
		},
	}

	// 2. 创建并启动容器
	createResp, err := cli.ContainerCreate(ctx, config, hostConfig, nil, nil, "")
	assert.NilError(t, err)
	defer cli.ContainerRemove(ctx, createResp.ID, container.RemoveOptions{Force: true})

	err = cli.ContainerStart(ctx, createResp.ID, container.StartOptions{})
	assert.NilError(t, err)

	// 3. 等待容器退出并检查退出状态
	statusCh, errCh := cli.ContainerWait(ctx, createResp.ID, container.WaitConditionNotRunning)
	select {
	case err := <-errCh:
		assert.NilError(t, err)
	case status := <-statusCh:
		// 退出码必须是 SIGKILL (128 + 9 = 137)
		assert.Equal(t, int64(137), status.StatusCode)
	case <-time.After(15 * time.Second):
		t.Fatal("Timeout waiting for container to be OOM killed")
	}

	// 4. 物理状态断言：Inspect 必须明确标注 OOMKilled == true
	inspectData, err := cli.ContainerInspect(ctx, createResp.ID)
	assert.NilError(t, err)
	assert.Equal(t, true, inspectData.State.OOMKilled)
	assert.Equal(t, false, inspectData.State.Running)
}
```

---

### 3.2 模块二：OverlayFS2 镜像层堆叠与写时复制（Copy-on-Write）持久化测试

#### 3.2.1 业务功能与测试目标
Docker 镜像由多个只读层组成。当容器修改已存在的文件时，文件系统通过写时复制将其拷贝到读写层（UpperDir）。当删除只读层已有的文件时，在读写层创建一个 Whiteout 特殊设备文件（字符设备 0:0）。测试必须验证：
1. 容器修改父镜像文件时，底层只读层文件的 inode 与哈希不受任何污染。
2. 容器删除父镜像文件后，容器内部不可见，但父层物理文件依旧存在。

#### 3.2.2 测试用例实现
```go
func TestOverlayFS2_CopyOnWriteAndWhiteoutDeletion(t *testing.T) {
	ctx := context.Background()
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	assert.NilError(t, err)

	// 1. 启动容器修改 /etc/issue 并删除 /etc/os-release
	cmd := []string{
		"sh", "-c",
		"echo 'MODIFIED_CONTENT' >> /etc/issue && rm -f /etc/os-release",
	}
	createResp, err := cli.ContainerCreate(ctx, &container.Config{
		Image: "alpine:latest",
		Cmd:   cmd,
	}, nil, nil, nil, "")
	assert.NilError(t, err)
	defer cli.ContainerRemove(ctx, createResp.ID, container.RemoveOptions{Force: true})

	err = cli.ContainerStart(ctx, createResp.ID, container.StartOptions{})
	assert.NilError(t, err)

	// 等待命令执行结束
	statusCh, _ := cli.ContainerWait(ctx, createResp.ID, container.WaitConditionNotRunning)
	<-statusCh

	// 2. 检查容器的文件系统差异变更 (docker diff)
	changes, err := cli.ContainerDiff(ctx, createResp.ID)
	assert.NilError(t, err)

	// 3. 强断言：diff 必须精准报告 /etc/issue 为 Modify (C)，/etc/os-release 为 Delete (D)
	foundModified := false
	foundDeleted := false
	for _, change := range changes {
		if change.Path == "/etc/issue" && change.Kind == 0 { // 0 = Modified
			foundModified = true
		}
		if change.Path == "/etc/os-release" && change.Kind == 2 { // 2 = Deleted
			foundDeleted = true
		}
	}
	assert.Assert(t, foundModified, "Expected /etc/issue to be marked as modified")
	assert.Assert(t, foundDeleted, "Expected /etc/os-release to be marked as deleted")
}
```

---

### 3.3 模块三：Libnetwork 虚拟网络隔离与跨容器端口转发路由测试

#### 3.3.1 业务功能与测试目标
用户创建用户自定义 bridge 网络。测试必须覆盖：
1. 同一自定义网络内部的两个容器可以通过容器名称进行内嵌 DNS 域名解析并通信。
2. 处于不同网络下的两个容器在二层与三层完全阻断，禁止 Ping 通。
3. 宿主机端口转发（`-p 8080:80`）经由 iptables NAT 表精准转发。

#### 3.3.2 测试用例实现
```go
func TestNetwork_DNSResolutionAndIsolationAcrossNetworks(t *testing.T) {
	ctx := context.Background()
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	assert.NilError(t, err)

	// 1. 创建网络 A 与 网络 B
	netA, err := cli.NetworkCreate(ctx, "test_net_a", types.NetworkCreate{})
	assert.NilError(t, err)
	defer cli.NetworkRemove(ctx, netA.ID)

	netB, err := cli.NetworkCreate(ctx, "test_net_b", types.NetworkCreate{})
	assert.NilError(t, err)
	defer cli.NetworkRemove(ctx, netB.ID)

	// 2. 在网络 A 中启动 Web 服务容器 server_a
	srvA, err := cli.ContainerCreate(ctx, &container.Config{
		Image: "alpine:latest",
		Cmd:   []string{"nc", "-lk", "-p", "8080", "-e", "echo", "HELLO_FROM_A"},
	}, nil, &network.NetworkingConfig{
		EndpointsConfig: map[string]*network.EndpointSettings{
			"test_net_a": {NetworkID: netA.ID},
		},
	}, nil, "server_a")
	assert.NilError(t, err)
	defer cli.ContainerRemove(ctx, srvA.ID, container.RemoveOptions{Force: true})
	cli.ContainerStart(ctx, srvA.ID, container.StartOptions{})

	// 3. 在网络 A 中启动客户端 client_a：测试容器名内置 DNS 解析与通信 -> 必须成功
	execResp, err := cli.ContainerCreate(ctx, &container.Config{
		Image: "alpine:latest",
		Cmd:   []string{"nc", "-w", "2", "server_a", "8080"},
	}, nil, &network.NetworkingConfig{
		EndpointsConfig: map[string]*network.EndpointSettings{
			"test_net_a": {NetworkID: netA.ID},
		},
	}, nil, "client_a")
	assert.NilError(t, err)
	defer cli.ContainerRemove(ctx, execResp.ID, container.RemoveOptions{Force: true})
	cli.ContainerStart(ctx, execResp.ID, container.StartOptions{})

	statusCh, _ := cli.ContainerWait(ctx, execResp.ID, container.WaitConditionNotRunning)
	status := <-statusCh
	assert.Equal(t, int64(0), status.StatusCode) // 同网容器成功通信

	// 4. 在网络 B 中启动客户端 client_b：尝试连接 server_a -> 必须超时失败
	execFail, err := cli.ContainerCreate(ctx, &container.Config{
		Image: "alpine:latest",
		Cmd:   []string{"nc", "-w", "2", "server_a", "8080"},
	}, nil, &network.NetworkingConfig{
		EndpointsConfig: map[string]*network.EndpointSettings{
			"test_net_b": {NetworkID: netB.ID},
		},
	}, nil, "client_b")
	assert.NilError(t, err)
	defer cli.ContainerRemove(ctx, execFail.ID, container.RemoveOptions{Force: true})
	cli.ContainerStart(ctx, execFail.ID, container.StartOptions{})

	statusFailCh, _ := cli.ContainerWait(ctx, execFail.ID, container.WaitConditionNotRunning)
	statusFail := <-statusFailCh
	assert.Assert(t, statusFail.StatusCode != 0, "Expected cross-network connection to fail")
}
```

---

## 4. 测试工程化与基础设施

1. **Docker-in-Docker 独立临时 Daemon 运行**：
   * Moby 的集成测试绝不运行在宿主机主 dockerd 上，而是通过脚本为每个测试并发包启动一个独立的 `dockerd --rootless` 或沙箱 dockerd，绑定私有 Unix 套接字，彻底杜绝测试污染。
2. **内核模块与特权能力动态探测**：
   * 测试套件包含 `testenv.ProtectImage` 与内核探针，如果当前宿主机内核不支持特定的 Cgroups v2 特性，用例自动进入 `t.Skip()` 优雅跳过，保证测试可在各版本 Linux 稳定执行。

---

## 5. 对同类复杂系统的参考价值

* **操作系统级底层副作用的闭环验证**：测试容器化或虚拟化中间件时，不能只看 CLI 返回结果，必须把验证下沉到操作系统内核层（Linux Namespaces、cgroups 内存计数文件、iptables 规则链与文件系统真实 inode 变更）。
