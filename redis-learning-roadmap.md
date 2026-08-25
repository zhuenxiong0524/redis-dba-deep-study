# Redis DBA 快速入门学习计划（面向 PG / Oracle DBA）

> 任务全集（按 PG 元数据格式）：`study_record/learning-roadmap.md`，已完成任务见 `study_record/completed/`。
>
> 目标读者：熟悉 PostgreSQL / Oracle 的 DBA，计划用约 6 周业余时间（每天 1~2 小时）快速建立 Redis 的
> 体系化认知，并具备独立运维、排障能力。整体思路：**用数据库 DBA 已有的心智模型迁移学习 Redis**，
> 每个阶段都配有"概念对照"和"动手实验"，避免只停留在命令记忆层面。

---

## 0. 心态转变（最重要的第一步）

| PostgreSQL / Oracle 心智 | Redis 心智 |
| --- | --- |
| 数据落盘，内存是缓存 | 数据全部在内存，磁盘（RDB/AOF）只是恢复手段 |
| 关系模型，SQL 查询 | Key-Value 模型，按 Key 直接访问，无查询优化器 |
| 多用户并发 + 锁 + MVCC | 单线程执行命令，天然无锁、命令原子 |
| 建表要规范化设计 | 设计围绕"访问路径"，一个场景一个 Key 结构 |
| 磁盘 IO、执行计划是性能核心 | 网络往返、内存占用、阻塞命令是性能核心 |

> 一句话：**Redis 不靠"查询"取数，靠"设计好的 Key"取数。** 容量规划以内存为核心。

---

## 1. 总览：6 周路线图

| 阶段 | 主题 | 关键产出 |
| --- | --- | --- |
| 第 1 周 | Redis 基础与数据模型 | 命令手册化 + 数据类型实战笔记 |
| 第 2 周 | 核心架构与内存管理 | 内存分析 + 淘汰策略实验 |
| 第 3 周 | 持久化机制 | RDB/AOF 原理 + 恢复演练 |
| 第 4 周 | 主从复制与 Sentinel | 高可用架构实验 |
| 第 5 周 | Cluster 集群 | 集群搭建与故障演练 |
| 第 6 周 | 监控排障与运维 | 故障演练 + 运维 checklist |

---

## 2. 第 1 周：基础与数据模型

### 知识点
- 部署：源码编译 / 官方二进制 / Docker，`redis-server`、`redis-cli`、`redis.conf` 关键配置项
- 五大基础类型：String、Hash、List、Set、ZSet
- 高级类型：Bitmap、HyperLogLog、Geo、Stream
- 通用命令：`KEYS`（生产禁用，用 `SCAN`）、`DEL`、`EXPIRE`/`TTL`、`TYPE`、`OBJECT ENCODING`
- 事务：`MULTI`/`EXEC`（无回滚、无隔离级别，对比 PG 事务的差异）
- 过期删除机制：惰性删除 + 定期删除（对比 PG 的 vacuum 概念迁移）

### 动手实验
- 用 `redis-cli` 完成 5 个场景：缓存、计数器、排行榜、去重、消息队列
- 对同一数据用不同类型实现，观察 `OBJECT ENCODING` 与内存差异

### 对照表（PG → Redis）
| PostgreSQL / Oracle | Redis |
| --- | --- |
| SQL | 命令集（`GET/SET/HSET/...`） |
| 表 / 行 | Key / Value |
| 主键 | Key（天然唯一） |
| 事务（ACID） | `MULTI/EXEC`（无回滚、无隔离） |
| 索引 | 由数据类型承担（ZSet 即"排序索引"） |
| 视图 | 无（数据冗余设计可接受） |

---

## 3. 第 2 周：核心架构与内存管理

### 知识点
- 单线程事件循环（Reactor 模型）、`io-threads`、阻塞命令的危害
- 对象编码：embstr / raw / intset / listpack / quicklist / skiplist
- 内存配置：`maxmemory`、8 种淘汰策略（`noeviction / allkeys-lru / volatile-lru / lfu ...`）
- 内存分析：`INFO memory`、`MEMORY USAGE`、`MEMORY DOCTOR`、内存碎片率（`mem_fragmentation_ratio`）
- 大 Key / 热 Key 概念（后面第 6 周深入）

### 动手实验
- 构造不同编码的对象，观察编码切换与内存变化
- 设置 `maxmemory` 为小值，分别用 LRU / LFU 策略触发淘汰，观察 `evicted_keys`
- 用 `redis-cli --bigkeys` 扫描大 Key

### 对照表
| PostgreSQL / Oracle | Redis |
| --- | --- |
| `shared_buffers` | `maxmemory`（但 Redis 所有数据都在内存） |
| 表膨胀 / 死元组 | 过期 Key 惰性删除（无版本链） |
| Buffer Cache 命中率 | `keyspace_hits / keyspace_misses` |
| 表空间大小估算 | `MEMORY USAGE` 单 Key 分析 |

---

## 4. 第 3 周：持久化机制

### 知识点
- RDB：`save`/`bgsave`、**fork + Copy-on-Write** 原理、对主线程的阻塞影响
- AOF：`appendfsync` 三种策略（always/everysec/no）、AOF 重写机制、AOF 损坏修复
- 混合持久化（Redis 7：RDB + AOF 增量）
- 持久化选型与故障恢复流程

### 动手实验
- 制造大内存数据集，执行 `bgsave`，观察 fork 耗时与内存上涨（COW）
- 配置 AOF 三种策略，对比 `fsync` 频率与性能
- 模拟故障：杀掉进程 → 分别用 RDB / AOF 恢复，验证数据完整性差异

### 对照表
| PostgreSQL / Oracle | Redis |
| --- | --- |
| WAL（预写日志） | AOF |
| checkpoint / 全量备份 | RDB 快照 |
| `pg_basebackup` | `BGSAVE` 生成 rdb 文件 |
| 崩溃恢复 replay WAL | 启动时加载 AOF / RDB |
| fork 后 COW | 与 PG `pg_ctl` fork 子进程类似心智 |

---

## 5. 第 4 周：主从复制与 Sentinel 高可用

### 知识点
- 主从复制原理：全量同步（RDB 传输）与部分同步（`psync` + repl backlog）
- 复制偏移量、`master_repl_offset`、延迟监控
- 主从配置：`replicaof`、`replica-read-only`、`repl-backlog-size`
- Sentinel：监控、通知、自动故障转移；`quorum` 与客观下线（ODOWN/SDOWN）
- 对比 PG：同步/异步流复制、`Patroni` 的选主逻辑

### 动手实验
- 搭建 1 主 2 从，验证全量/增量同步，观察 `INFO replication`
- 制造网络延迟，观察复制延迟指标
- 搭建 3 节点 Sentinel，手动 kill 主节点，观察自动 failover 全流程

### 对照表
| PostgreSQL / Oracle | Redis |
| --- | --- |
| 流复制 / standby | 主从复制 / replica |
| 同步提交 `synchronous_commit` | `WAIT` 命令（少数场景） |
| Patroni / etcd 选主 | Sentinel（主观/客观下线 + 投票） |
| `pg_stat_replication` | `INFO replication` |
| 复制延迟 | `master_repl_offset - slave_repl_offset` |

---

## 6. 第 5 周：Cluster 集群

### 知识点
- 分片原理：16384 个 hash slot，`CRC16(key) % 16384`
- 集群拓扑：集群总线（Cluster Bus）、Gossip 协议、心跳
- 集群搭建：`redis-cli --cluster create`、`add-node`、`reshard`、`rebalance`
- 故障转移：主节点失联 → 从节点自动提升
- 集群限制：多 Key 操作需同槽（hash tag）、`pipeline`/事务的槽约束、客户端路由（MOVED/ASK）

### 动手实验
- 搭建 3 主 3 从集群，验证数据均匀分布
- 执行 `reshard` 迁移槽位，观察数据迁移过程
- kill 一个主节点，观察从节点提升；再恢复原主，观察角色变化

### 对照表
| PostgreSQL / Oracle | Redis |
| --- | --- |
| 分库分表 / 分区表 | hash slot 分片 |
| 分区键选择 | 业务 Key 的 hash 设计（避免热点槽） |
| 分布式事务 | 无跨节点事务（同槽才支持） |
| 扩容缩容 | `reshard` 在线迁移槽位 |

---

## 7. 第 6 周：监控、排障与运维

### 知识点
- 监控指标：`INFO` 关键项、`redis-cli --stat`、`SLOWLOG`、`LATENCY` 监控、`MONITOR`（慎用）
- 大 Key / 热 Key 的发现与治理
- 高频故障模式：
  - fork 阻塞（bgsave / AOF rewrite）
  - 内存淘汰风暴（evicted_keys 飙升）
  - 阻塞命令（`KEYS`、`SMEMBERS`、大 `DEL`、`SORT`）
  - 缓存雪崩 / 击穿 / 穿透
- 备份与恢复：RDB 定期备份、AOF 备份、异地备份
- 安全：ACL（Redis 6+）、`requirepass`、TLS、`rename-command` 禁用危险命令
- 版本升级与 `redis-check-aof` / `redis-check-rdb` 工具

### 动手实验
- 故意制造大 Key + 慢查询，用 `SLOWLOG GET` 定位
- 模拟缓存穿透场景，设计并验证解决方案（空值缓存 / 布隆过滤器）
- 做一次完整的备份 → 恢复演练

### 对照表
| PostgreSQL / Oracle | Redis |
| --- | --- |
| `pg_stat_statements` / 慢查询日志 | `SLOWLOG` |
| `pg_stat_activity` 活跃会话 | `CLIENT LIST` |
| autovacuum 阻塞排查 | fork / COW 阻塞排查 |
| 权限管理（角色 / 用户） | ACL（用户 + 命令 + key 权限） |
| `pg_dump` 备份 | `BGSAVE` / `redis-cli --rdb` |
| 连接池 pgBouncer | 客户端连接池（server 侧无连接池概念） |

---

## 8. 进阶方向（6 周之后，按需选择）

1. **缓存架构设计**：Cache-Aside / Read-Through / Write-Through、缓存一致性（对比 PG 没有对应概念）
2. **分布式锁**：SETNX + Lua 实现、Redlock 的争议与适用边界
3. **Lua 脚本与 Redis Functions**：原子性组合操作
4. **Redis 7 / 8 新特性**：Redis Functions、Sharded Pub/Sub、ACL v2、`CLIENT SETINFO`
5. **源码阅读（用学 PG 源码的方法）**：event loop（ae.c）、dict 字典、ziplist/listpack、skiplist、
   RDB/AOF 序列化——这是 DBA 深水区，建议先把前面 6 周走完
6. **替代品对比**：Redis vs Memcached、Redis vs Valkey（fork 分支）、云厂商托管版差异
   （阿里云 Tair、AWS ElastiCache）

---

## 9. 推荐资料

### 文档
- Redis 官方文档：https://redis.io/docs/ （重点：数据类型、持久化、复制、集群、ACL）
- 自带 `redis.conf` 注释版（每装一次 Redis 就是一本手册）

### 书
- 《Redis 开发与运维》付磊 / 张益军 —— **DBA 视角最对口，强烈推荐**
- 《Redis 设计与实现》黄健宏 —— 源码实现细节，适合进阶

### 实验环境
- 本机 `make install` 编译安装，或 Docker：`docker run -d --name redis -p 6379:6379 redis:7`
- 学习期间建议至少搭一次：单机 → 主从 + Sentinel → 3主3从 Cluster

---

## 10. 学习方法建议

1. **迁移式学习**：每个新概念先问"PG/Oracle 里对应的是什么？"，没有对应物（如淘汰策略、单线程）
   的就是 Redis 特色，重点记
2. **每阶段留产出物**：一篇实验笔记（环境、现象、结论），6 周后你有 6 篇可复用的笔记
3. **命令不要背**：用 `redis-cli` 手敲 + `help @string` 这类命令手册查询即可
4. **多制造故障**：kill -9、拔网络、写满内存——DBA 的肌肉记忆来自演练
5. **关注版本**：当前主线是 Redis 7.x / 8.x（8.0 已发布），网上老教程多是 5/6 的写法，注意甄别
