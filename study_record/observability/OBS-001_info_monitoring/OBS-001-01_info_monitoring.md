# OBS-001-01 INFO 指标与监控体系

> Redis 8.6.2 / 127.0.0.1:6379 / 密码 123456
>
> 本文是"监控第一课"：**先学会让 Redis 自己"说话"（INFO），再谈监控系统**。
> 所有数值均在本机实测（证据：`evidence/info-sections.json`、`metrics-glossary.json`、`benchmark.json`、`stat-output.json`、`pg-mapping.json`）。

---

## 0. 一句话心法

> **监控 Redis 不是装个面板，而是先读懂 INFO 这一个命令输出的 14 个区块。**
> 面板上 90% 的指标，都是从 INFO 里"翻译"过去的。

---

## 1. 环境准备

```bash
# 交互式查看（-n 5 用逻辑库 DB5，不碰业务数据）
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning -n 5

# 非交互取全部区块
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning INFO

# 只取某个区块（Server / Clients / Memory / Stats / Keyspace ...）
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning INFO stats
```

- `--no-auth-warning`：本实例启用了 requirepass，避免每次打印"密码暴露在命令行"的警告（生产用 `REDISCLI_AUTH` 环境变量更安全）；
- 本实例 `CONFIG` 命令被 `rename-command` 禁用，所以**不能用 CONFIG 改参数**，但 INFO 是只读统计，完全不受影响。

---

## 2. INFO 的 14 个区块（8.6.2 实测）

8.x 相比 7.x 新增了 **Hotkeys**（热点 key）与 **Keysizes**（key 大小分布）两个区块，实测共 14 个：

```text
Server / Clients / Memory / Persistence / Threads / Stats /
Replication / CPU / Hotkeys / Modules / Errorstats / Cluster /
Keyspace / Keysizes
```

每个区块只负责一类信息，读的时候"按需取块"：

| 区块 | 回答什么问题 | 关键指标（实测值） |
|---|---|---|
| Server | 这是个什么实例？ | `redis_version=8.6.2`、`redis_mode=standalone`、`hz=10`、`config_file=/data/redis/conf/redis.conf` |
| Clients | 谁连着？卡住没？ | `connected_clients=2`、`blocked_clients=0`、`maxclients=10000` |
| Memory | 内存用了多少、会不会被淘汰？ | `used_memory_human=3.26M`、`used_memory_peak_human=5.64M`、`maxmemory_human=1.25G`、`maxmemory_policy=volatile-lru`、`mem_fragmentation_ratio=4.88` |
| Persistence | 数据落盘正常吗？ | `aof_enabled=1`、`rdb_last_bgsave_status=ok`、`aof_last_write_status=ok`、`latest_fork_usec=394` |
| Threads | 线程模型（8.x 命令执行仍是单线程） | `io_threads` 相关配置 |
| Stats | 吞吐/命中/过期/驱逐（**最常用**） | `total_commands_processed=957762`、`instantaneous_ops_per_sec=0`、`keyspace_hits=170358`、`keyspace_misses=26`、`expired_keys=5005`、`evicted_keys=0` |
| Replication | 主从健康吗？ | `role=master`、`connected_slaves=0` |
| CPU | CPU 烧在哪？ | `used_cpu_user/sys` 累计秒数 |
| Hotkeys | 哪些 key 被疯狂访问？（8.0+） | 默认空，需配置热点采样 |
| Modules | 加载了什么模块？ | 本机无模块 |
| Errorstats | 客户端错在哪？（8.x 新增） | `ERR=29`、`EXECABORT=1`、`NOAUTH=3`、`WRONGPASS=1`、`WRONGTYPE=1` |
| Cluster | 集群模式/槽位 | `cluster_enabled=0` |
| Keyspace | 每个库有多少 key？ | `db0:keys=1`、`db5:keys=4`、`db15:keys=0` |
| Keysizes | key 大小分布（8.0+） | 默认空，需配置采样 |

> **心法：日常巡检只要盯住三个区块——Memory、Stats、Replication。**
> 其余区块属于"出问题时才去翻"的取证材料。

---

## 3. 关键指标含义与判断标准（初学者版）

上表只说了"有哪些指标、实测多少"，这一节补上最重要的部分：**每个指标到底是什么意思、怎么判断好坏**。
（完整速查表见证据 `evidence/metrics-glossary.json`。）

### 3.1 Memory 区块：内存是 Redis 的命根子

| 指标（单位） | 含义 | 怎么判断 |
|---|---|---|
| `used_memory`（字节） | 分配器实际使用的内存（数据+内部开销） | 核心水位指标：超过 `maxmemory` 的 **70%** 就要关注，逼近 100% 会触发淘汰 |
| `used_memory_rss`（字节） | 操作系统看到的常驻内存（含 jemalloc 元数据与碎片） | 明显大于 `used_memory` = 碎片/分配器开销大 |
| `used_memory_peak`（字节） | 历史最高水位（重启清零） | 比当前大很多 = 曾有大流量或大 key；用于规划要不要扩容 |
| `maxmemory`（字节）+ `maxmemory_policy` | 内存上限；达到后按策略淘汰 key | 本机 `volatile-lru` = 只淘汰**带 TTL 的 key** 中最久未用的；若策略是 `noeviction`，内存满时写命令直接报错 |
| `mem_fragmentation_ratio`（比值） | `used_memory_rss / used_memory`，内存碎片率 | 正常 **1.0~1.5**；>1.5 碎片偏高（本机 4.88 属偏高，多为 jemalloc 统计口径 + THP 透明大页影响，先看 rss 实际占用）；**<1 说明可能用了 swap，很危险** |

> **心法：`evicted_keys` 只要 >0，就意味着内存已满、开始丢数据（按策略丢，可能丢热数据）。这是最高优先级告警。**

### 3.2 Stats 区块：吞吐与命中（最常用）

| 指标（单位） | 含义 | 怎么判断 |
|---|---|---|
| `total_commands_processed`（次） | 自启动累计处理的命令数（计数器） | 看**增量**：两次采样差值/秒 = 实际吞吐 |
| `instantaneous_ops_per_sec`（次/秒） | 当前 1 秒内执行的命令数（瞬时 OPS） | 最直观的负载值（本机空闲=0，压测时 3.5 万~4 万）；与历史峰值对比判断是否突发 |
| `keyspace_hits` / `keyspace_misses`（次） | key 命中/未命中累计 | 命中率 = `hits/(hits+misses)`；本机 170358/(170358+26)≈**99.98%** 健康；明显下降 = 过期、穿透或冷数据 |
| `expired_keys`（次） | 累计被过期清理的 key 数 | 瞬间暴涨 = 大批 key 同时过期（**雪崩前兆**），检查 TTL 是否扎堆 |
| `evicted_keys`（次） | 因 maxmemory 被淘汰驱逐的 key 数 | **>0 就必须处理**（本机=0）：扩容、优化 key、调淘汰策略 |
| `total_net_input/output_bytes`（字节） | 累计网络入/出流量 | 增量反映带宽；配合瞬时 kbps 判断是否打满网卡 |
| `rejected_connections`（次） | 因超过 maxclients 被拒绝的连接 | >0 = 连接数打满，客户端连不上，升 maxclients 或查连接泄漏 |

### 3.3 Clients / Replication / Persistence / CPU / Errorstats

| 区块 | 指标 | 含义 → 判断 |
|---|---|---|
| Clients | `connected_clients` | 当前连接数；接近 `maxclients=10000` 告警（先排除监控/压测连接） |
| Clients | `blocked_clients` | 挂在 `BLPOP` 等阻塞命令上的客户端；持续上涨 = 队列积压，业务在排队 |
| Clients | `client_recent_max_input/output_buffer` | 单连接最大缓冲；过大 = 有客户端发超大命令/读超大 key |
| Replication | `role` / `connected_slaves` | 角色与已连从库数；从库少了 = 复制断链 |
| Replication | `master_repl_offset`（主从偏移量） | 与从库 offset 的差值 = 复制延迟，持续拉大 = 从库落后 |
| Persistence | `rdb_last_bgsave_status` / `aof_last_write_status` | 最近一次落盘结果；`err` = 落盘失败，立刻查磁盘/权限 |
| Persistence | `latest_fork_usec`（微秒） | 最近一次 fork 耗时；fork 期间可能阻塞命令，本机 394us 正常，**秒级要查内存量与 THP** |
| CPU | `used_cpu_sys/user`（秒） | 累计内核/用户态 CPU 时间；采样差值高 = CPU 忙，查慢命令、过期清理、持久化 |
| Errorstats | `ERR / WRONGTYPE / NOAUTH / WRONGPASS / EXECABORT`（次） | 按错误码聚合的错误计数：`WRONGTYPE` = 对错误类型做操作（业务 bug）；`NOAUTH/WRONGPASS` = 认证失败（配置错或攻击）；`EXECABORT` = 事务被取消；`ERR` = 参数/用法错误（本机 ERR=29） |
| Keyspace | `keys / expires / avg_ttl` | 各库 key 总数 / 带 TTL 的 key 数 / 平均剩余 TTL；`keys` 突增突降 = 批量写入或大 key 删除；`avg_ttl` 归零 = 全部要过期 |

### 3.4 新手 5 个"一看就懂"的红线

```text
1. evicted_keys > 0            → 内存已满，正在丢数据（最高优先级）
2. used_memory 逼近 maxmemory  → 内存告警，准备扩容/瘦身
3. mem_fragmentation_ratio>1.5 → 碎片偏高，评估重启/调 jemalloc
4. keyspace 命中率明显下降     → 缓存失效/穿透，业务受影响
5. latest_fork_usec 达秒级     → fork 阻塞风险，查 THP 与内存量
```

---

## 4. 实战：负载前后指标变化（redis-benchmark）

空跑 INFO 只能看到"静态基线"，真正的监控价值是**对比**。用压测制造负载，看指标怎么动：

```bash
# 4 类命令 × 10 万请求 × 50 并发，隔离库 DB15（不污染业务库）
redis-benchmark -u redis://default:123456@127.0.0.1:6379/15 \
  -t set,get,incr,lpush -n 100000 -c 50 -q
```

**坑（实测）**：URI 里必须写 `default:` 用户名，写成 `redis://:pass@` 会报 `WRONGPASS`。
**预期警告**：`Could not fetch server CONFIG`——实例禁用了 CONFIG，不影响压测结果，属预期现象。

压测结果（本机 10 万级请求/类型）：

| 命令 | 吞吐（rps） | p50 延迟 |
|---|---|---|
| SET | ≈35,000 | 1.09 ms |
| GET | ≈38,000 | 0.74 ms |
| INCR | ≈40,000 | 0.92 ms |
| LPUSH | ≈33,000 | 1.10 ms |

负载中的对比变化（INFO stats 前后采样）：

```text
total_commands_processed : 376,819 → 777,224（+40 万）
keyspace_hits            : 增量约 +10 万（压测读取全部命中）
total_net_input_bytes    : 增量约 +15.8 MB
instantaneous_ops_per_sec: 压测中约 35,000～40,000，空闲归 0
```

> **心法：只看瞬时值没意义，要看"涨跌方向 + 量级"。**
> 比如 `instantaneous_ops_per_sec` 从 0 冲到 4 万，说明压测/突发流量真实到达；之后再归 0，说明负载已退去。

---

## 5. redis-cli --stat：一行一个时刻的"轻量监控"

不想进交互模式、只想看动态趋势？`--stat` 每秒打印一行采样：

```bash
stdbuf -oL redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning --stat
```

实测输出（配合后台压测）：

```text
CONFIG GET databases fails: ERR unknown command 'CONFIG' ...
use default value 16 instead
------- data ------ --------------------- load -------------------- - child -
keys       mem      clients blocked requests            connections
6          4.13M    52      0       953228 (+0)         3803
6          3.25M    2       0       957754 (+4526)      3803
6          3.25M    2       0       957755 (+1)         3803
```

- 列含义：`keys`=当前库 key 数，`mem`=内存，`clients`=连接数，`blocked`=阻塞客户端，`requests`=累计命令数（`(+N)` 是每秒增量），`connections`=累计连接数；
- 第一行 `clients=52` 正是压测的 50 并发 + 采样连接，压测结束后回落到 2——一眼看出负载进出；
- **注意**：启动时会先打印一行 `CONFIG GET databases fails: ERR unknown command 'CONFIG' ... use default value 16 instead`。这是实例 `rename-command` 禁用了 CONFIG 导致的，**属预期现象，不影响统计采样**（8.x 下看到不要慌）。

> **心法：`--stat` 就是 Redis 版的 `watch -n1`，适合"盯 30 秒"的快速体检；长期监控请交给 Prometheus 等拉取 INFO 解析。**

---

## 6. 与 PG 监控指标对照（双数据库统一口径）

| Redis（INFO） | PG（18.4 对应视图） | 说明 |
|---|---|---|
| `Stats: total_commands_processed` / `instantaneous_ops_per_sec` | `pg_stat_database: xact_commit+xact_rollback` / 活跃会话 | 吞吐与负载 |
| `Stats: keyspace_hits` / `keyspace_misses` | `pg_statio_user_tables: heap_blks_hit` / `heap_blks_read` | 命中率类指标 |
| `Clients: connected_clients` / `blocked_clients` | `pg_stat_activity: state='active' 计数` / `wait_event` | 连接与会话 |
| `Memory: used_memory` / `maxmemory` / `mem_fragmentation_ratio` | `shared_buffers` / `work_mem` / `pg_stat_bgwriter` | 内存水位 |
| `Replication: role` / `connected_slaves` / 偏移量 | `pg_stat_replication: state` / `sent_lsn` / `replay_lsn` | 主从状态与延迟 |
| `Persistence: rdb_last_bgsave_status` / `aof_last_write_status` / `latest_fork_usec` | `pg_stat_bgwriter` checkpoint 相关 | 落盘与 checkpoint |
| `Errorstats: 错误码计数` | 错误日志 / 异常扫描统计 | 错误与异常 |
| `SLOWLOG GET`（延迟类，见 OBS-002） | `pg_stat_statements: mean_exec_time` / `max_exec_time` | 慢请求定位 |

> **心法：PG 看"库和表"，Redis 看"实例和 key"；但监控体系的四类指标——吞吐、命中、内存、复制——两边口径完全对得上。**

---

## 7. 小结

- INFO 一次返回 **14 个区块**（8.x 在 7.x 基础上新增 Hotkeys / Keysizes），按需取块、盯住 Memory / Stats / Replication 三个核心；
- 指标含义速查：内存看 `used_memory/maxmemory/evicted_keys/碎片率`，吞吐看 `total_commands/instantaneous_ops`，健康看 `命中率/复制偏移量/落盘状态`，红线是 **`evicted_keys>0`、内存逼近上限、碎片率>1.5、命中率下降、fork 秒级**；
- 压测对比证实：`total_commands_processed`、`keyspace_hits`、网络字节数与 OPS 同步反映负载，**监控要读变化量而非单点值**；
- `redis-cli --stat` 每秒一行，适合快速体检；CONFIG 被禁用的实例会打印警告行，属预期；
- 与 PG 对照后，两边监控口径可统一：吞吐↔`pg_stat_database`、命中↔`pg_statio_*`、连接↔`pg_stat_activity`、复制↔`pg_stat_replication`。

**后续深化**：慢命令与延迟定位进入 **OBS-002 SLOWLOG 与延迟分析**；内存碎片与淘汰策略深挖进入 MEM-001。
