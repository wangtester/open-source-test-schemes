# Redis 自动化测试方案与底层存储用例全集深度剖析

## 1. 项目背景与系统架构全景

[Redis](https://github.com/redis/redis) 是全球最高性能的内存分布式键值存储系统。系统采用纯 C 语言编写，核心依托单线程事件循环驱动（基于 epoll / kqueue / select 的高性能 I/O 多路复用器 ae 模块）。系统具备丰富的高级数据结构（Strings, Hashes, Lists, Sets, Sorted Sets, HyperLogLog, Geospatial, Streams），以及 AOF/RDB 双持久化机制、主从增量同步（PSYNC）与分布式集群（Redis Cluster）。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Redis Clients / SDKs                            │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ RESP 协议 (REdis Serialization Protocol)
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Redis Server Core (C / Single Thread Event Loop)    │
├───────────────────────────────┬────────────────────────────────────────┤
│ ae I/O Multiplexing Loop      │ Dict & SDS Memory Allocator (Jemalloc) │
│ - 文件事件 (Socket 读写监听)  │ - 渐进式 Rehash 扩容防卡顿             │
│ - 时间事件 (serverCron 后台)  │ - 内存碎片整理与 Maxmemory 淘汰算法    │
├───────────────────────────────┼────────────────────────────────────────┤
│ Persistence Subsystem         │ Replication & Cluster Subsystem        │
│ - RDB 快照 (fork + 写时复制)  │ - PSYNC2 增量全量自适应主从复制        │
│ - AOF 日志 (appendfsync 刷新) │ - 16384 Hash Slot 动态迁移与重定向     │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Redis 拥有极其著名的原生 **TCL（Tool Command Language）自动化测试框架**（`tests/` 目录），能够无侵入地控制 Redis 进程拓扑、断电模拟与毫秒级时序：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Unit Tests (TCL / C)** | SDS 动态字符串、Dict 渐进式扩容、ZipList/QuickList 紧凑编码 | Redis TCL Test Suite | 纯内存独立 Redis 临时单实例 |
| **Persistence Tests** | AOF Rewrite 文件压缩、RDB 快照一致性、Crash 掉电后恢复 | TCL + `kill -9` 信号模拟 | 独立的临时数据文件目录 (`/tmp/redis-test`) |
| **Replication Integration** | Master/Replica 链路断开重连、PSYNC 积压缓冲区溢出降级 | 多进程 Redis 拓扑驱动 | 独立端口（6379, 6380...）进程树管理 |
| **Cluster Dynamic Failover** | 槽位迁移、MOVED / ASK 转向处理、半数节点失联自动选举 | 6+ 节点 Redis Cluster TCL 编排 | 独立的 Cluster 配置文件集群 |
| **Transaction & Lua Sandbox** | MULTI/EXEC 事务原子性、Lua 脚本沙箱禁止非确定性全局状态 | TCL RESP 客户端调用 | 每套用例前执行 `FLUSHALL` |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：AOF/RDB 持久化在崩溃（Crash）场景下的物理恢复测试

#### 3.1.1 业务功能与测试目标
当系统配置 `appendfsync always` 时，每次写命令必须立即刷盘。此时若服务器遭遇硬件断电（在测试中使用 `kill -9` 强杀 Redis 进程），重新拉起服务后，数据必须 100% 完整保留，且未完成写入的残缺尾部数据能通过 `redis-check-aof --fix` 自动修复。

#### 3.1.2 测试用例实现（Redis 原生 TCL 测试脚本）
```tcl
start_server {tags {"aof"} overrides {appendonly yes appendfsync always}} {
    set master [srv 0 client]
    set master_host [srv 0 host]
    set master_port [srv 0 port]
    set aof_file [file join [srv 0 stdout] "appendonly.aof"]

    test {AOF durability: survives crash after SET commands} {
        # 1. 连续写入 1000 个键值对
        for {set j 0} {$j < 1000} {incr j} {
            $master set "key:$j" "val:$j"
        }

        # 2. 模拟服务器宿主机物理掉电：不发送 SHUTDOWN，直接发送 SIGKILL 强杀进程
        set pid [srv 0 pid]
        exec kill -9 $pid

        # 3. 验证文件物理落盘：AOF 文件大小必须大于 0
        assert {[file size $aof_file] > 0}

        # 4. 重启全新的 Redis 实例，加载此 AOF 文件
        restart_server 0 1 1
        set recovered_client [srv 0 client]

        # 5. 校验数据恢复率：1000 个 Key 必须一个不丢
        set missing_keys 0
        for {set j 0} {$j < 1000} {incr j} {
            if {[$recovered_client get "key:$j"] ne "val:$j"} {
                incr missing_keys
            }
        }
        assert_equal 0 $missing_keys
    }
}
```

---

### 3.2 模块二：主从全量与增量复制（PSYNC2）断线重连与积压缓冲区（Backlog）测试

#### 3.2.1 业务功能与测试目标
当 Replica（从节点）与 Master 发生网络闪断时，若从节点的复制偏移量（Offset）仍在 Master 的 `repl-backlog` 范围内，重新连接后只发送增量数据（PSYNC），不需要触发耗时的全量 RDB 传输。

#### 3.2.2 测试用例实现（TCL 测试）
```tcl
start_server {tags {"repl"} overrides {repl-backlog-size 1mb}} {
    set master [srv 0 client]
    set master_port [srv 0 port]

    start_server {tags {"repl"}} {
        set slave [srv 0 client]
        
        test {Replication: Partial Resynchronization (PSYNC) succeeds after short disconnection} {
            # 1. 将 slave 挂载到 master
            $slave replicaof 127.0.0.1 $master_port
            wait_for_condition 50 100 {
                [status $slave role] eq "slave" && [status $slave master_link_status] eq "up"
            } else {
                fail "Slave failed to connect to master"
            }

            # 2. 写入基准数据并记录初始偏移量
            $master set key1 value1
            wait_for_condition 50 100 {
                [$slave get key1] eq "value1"
            } else {
                fail "Replication lag"
            }
            set initial_sync_count [status $master sync_full]

            # 3. 模拟网络闪断：暂停从节点的网络连接 (使用 CLIENT PAUSE 或停止 Slave 进程)
            pause_process [srv 0 pid]

            # 4. 在 Master 上继续产生增量写入
            for {set j 0} {$j < 100} {incr j} {
                $master set "burst:$j" "val"
            }

            # 5. 恢复 Slave 进程继续运行
            resume_process [srv 0 pid]

            # 6. 等待同步追平
            wait_for_condition 50 100 {
                [$slave get "burst:99"] eq "val"
            } else {
                fail "Slave failed to sync burst data"
            }

            # 7. 强断言：Full Sync 全量同步次数绝对不能增加，必须走 PSYNC 增量
            set current_sync_count [status $master sync_full]
            assert_equal $initial_sync_count $current_sync_count
        }
    }
}
```

---

### 3.3 模块三：Redis Cluster 槽位（Slots）动态迁移与 MOVED / ASK 重定向测试

#### 3.3.1 业务功能与测试目标
Redis 集群拥有 16384 个哈希槽。当执行集群扩缩容迁移 Slot 时：
1. 处于迁移中的键访问触发 `ASK` 临时重定向。
2. 迁移完成的键访问触发 `MOVED` 永久重定向，客户端自动更新路由表。

#### 3.3.2 测试用例实现
```tcl
test {Redis Cluster: Slot migration triggers ASK then MOVED redirect} {
    set node_a [get_cluster_node 0]
    set node_b [get_cluster_node 1]

    # 1. 槽位 100 初始在 Node A 上
    $node_a cluster addslots 100
    set target_key "test_key_{100}" ;# 确保 CRC16 映射到 100 号槽位
    $node_a set $target_key "original_val"

    # 2. 将 100 号槽位设置为迁移状态 (MIGRATING on A, IMPORTING on B)
    $node_b cluster setslot 100 importing [$node_a cluster myid]
    $node_a cluster setslot 100 migrating [$node_b cluster myid]

    # 3. 将物理 Key 迁移至 Node B
    $node_a migrate [srv -1 host] [srv -1 port] "" 0 5000 keys $target_key

    # 4. 客户端再次访问 Node A：必须收到 ASK 临时重定向响应
    catch {$node_a get $target_key} err
    assert_match {ASK 100 *} $err

    # 5. 正式将槽位所有权彻底变更为 Node B
    $node_a cluster setslot 100 node [$node_b cluster myid]
    $node_b cluster setslot 100 node [$node_b cluster myid]

    # 6. 再次访问 Node A：必须收到 MOVED 永久重定向
    catch {$node_a get $target_key} err_moved
    assert_match {MOVED 100 *} $err_moved
}
```

---

## 4. 测试工程化与基础设施

1. **确定性多实例进程树生命周期**：
   * Redis TCL 测试引擎实现了 `start_server` 与 `restart_server` 原语，每个测试块自动分配随机端口、临时配置与独立日志路径，无论测试通过还是异常中断，顶层退出处理器均确保 `kill` 掉所有子进程。
2. **极端内存压力与淘汰算法验证**：
   * 测试套件支持设置 `--maxmemory 1mb --maxmemory-policy allkeys-lru`，并并发注入 10MB 数据，验证物理内存使用始终稳定在上限之内。

---

## 5. 对同类复杂系统的参考价值

* **掉电与崩溃模拟的标准规范**：测试高可靠存储系统时，不能依赖温和的 `exit(0)`，必须在事务中途、持久化刷盘中途注入 `kill -9` 甚至物理坏块，才能验证 WAL / AOF 的真正自愈力。
