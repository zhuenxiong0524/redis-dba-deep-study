# Redis DBA 第一年学习总任务

> 版本：v0.1
> 任务基线：Redis 8.6.2（本机实例：127.0.0.1:6379，见 ENV-001）
> 核心任务：25 项（初版，随学习推进扩展）
> 任务完成定义：当前专题计划中的系列文章达到对应文章规范并完成质量检查。

## 使用原则

- 本文是全年任务全集，不按月份锁死任务数量。
- 详细学习计划（6 周阶段划分、PG/Oracle ↔ Redis 概念对照表、资料清单）见根目录 `redis-learning-roadmap.md`。
- 熟悉内容可以快速形成文章并完成；陌生核心机制优先进入源码与实验验证。
- `系列已完成` 不等于永久掌握，文章允许持续修订。
- 任务元数据格式沿用 PostgreSQL 学习仓库的解析协议。

## 权重与积分

- `S`：快速整理，1 分
- `M`：标准研究，2 分
- `L`：深度机制，3 分
- `XL`：综合系列或项目，5 分

## 当前任务

- 当前进行中任务由专题 `.idx.md` 的 `系列状态` 决定。
- 初次运行建议从 `ENV-001` 开始；已有成果可直接验证、成文并完成。

## ENV 环境、编译与调试

### ENV-001 Redis 8.6.2 源码获取、编译与安装

- 年度属性：核心
- 权重：S
- 积分：1
- 优先级：P0
- 前置任务：无
- 主文章类型：environment
- 研究深度：快速沉淀
- 任务说明：源码目录、实例目录、redis.conf 关键项、systemd 托管、功能验证、与 PG 环境对照

### ENV-002 实例生命周期与多实例管理

- 年度属性：核心
- 权重：S
- 积分：1
- 优先级：P1
- 前置任务：ENV-001
- 主文章类型：environment
- 研究深度：快速沉淀
- 任务说明：启动/停止/重启/平滑下线、多端口多实例、systemd 模板、日志轮转

## DAT 数据模型与命令

### DAT-001 五大基本类型与内部编码

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P0
- 前置任务：ENV-001
- 主文章类型：datatype
- 研究深度：标准研究
- 任务说明：String/Hash/List/Set/ZSet 命令矩阵与 `OBJECT ENCODING` 编码切换实验

### DAT-002 高级类型：Bitmap / HyperLogLog / Geo / Stream

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P1
- 前置任务：DAT-001
- 主文章类型：datatype
- 研究深度：标准研究
- 任务说明：应用场景、内存优势、命令实战（UV 统计、签到、延迟队列、消息）

### DAT-003 过期、TTL 与键删除机制

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P1
- 前置任务：DAT-001
- 主文章类型：datatype
- 研究深度：标准研究
- 任务说明：惰性删除 + 定期删除源码路径、`expired_keys` 统计、与 PG vacuum 心智对照

### DAT-004 事务、管道与 Lua 脚本

- 年度属性：核心
- 权重：S
- 积分：1
- 优先级：P1
- 前置任务：DAT-001
- 主文章类型：datatype
- 研究深度：快速沉淀
- 任务说明：MULTI/EXEC 无回滚语义、WATCH 乐观锁、pipeline 性能、与 PG 事务对比

## ARC 核心架构

### ARC-001 单线程事件循环与 I/O 多路复用

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P1
- 前置任务：DAT-001
- 主文章类型：architecture
- 研究深度：深度机制
- 任务说明：ae.c 事件循环、epoll、命令处理主流程、io-threads、阻塞命令影响

### ARC-002 对象系统与内存编码实现

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P1
- 前置任务：DAT-001
- 主文章类型：architecture
- 研究深度：深度机制
- 任务说明：robj、引用计数、embstr/intset/listpack/quicklist/skiplist 结构

### ARC-003 阻塞命令与延迟问题

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P2
- 前置任务：ARC-001
- 主文章类型：architecture
- 研究深度：标准研究
- 任务说明：KEYS/SMEMBERS/SORT/大 DEL 等阻塞场景、命令耗时与 `commandstats`

## MEM 内存管理

### MEM-001 maxmemory 与淘汰策略

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P1
- 前置任务：ARC-002
- 主文章类型：memory
- 研究深度：标准研究
- 任务说明：8 种策略、近似 LRU/LFU、`evicted_keys` 淘汰风暴实验

### MEM-002 内存分析与碎片治理

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P2
- 前置任务：MEM-001
- 主文章类型：memory
- 研究深度：标准研究
- 任务说明：INFO memory、MEMORY USAGE/DOCTOR、碎片率、jemalloc 特性、容量规划

## PER 持久化

### PER-001 RDB 快照机制与 fork/COW

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P1
- 前置任务：ARC-001
- 主文章类型：persistence
- 研究深度：深度机制
- 任务说明：save/bgsave 触发、fork+COW 内存放大、`rdb_changes_since_last_save`、阻塞分析

### PER-002 AOF 与混合持久化

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P1
- 前置任务：PER-001
- 主文章类型：persistence
- 研究深度：深度机制
- 任务说明：appendfsync 三种策略、AOF 重写、base+incr 文件、崩溃恢复流程、与 PG WAL 对照

### PER-003 备份与恢复演练

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P2
- 前置任务：PER-002
- 主文章类型：persistence
- 研究深度：标准研究
- 任务说明：BGSAVE 备份、redis-check-rdb/aof 修复、点表恢复演练、与 pg_basebackup 对照

## REP 复制与高可用

### REP-001 主从复制原理：全量与部分同步

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P1
- 前置任务：PER-001
- 主文章类型：replication-ha
- 研究深度：深度机制
- 任务说明：replicaof、psync、repl backlog、复制偏移量与延迟、与 PG 流复制对照

### REP-002 Sentinel 高可用架构

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P2
- 前置任务：REP-001
- 主文章类型：replication-ha
- 研究深度：深度机制
- 任务说明：监控/通知/故障转移、quorum、SDOWN/ODOWN、客户端发现、与 Patroni 对照

## CLU 集群

### CLU-001 Cluster 分片：槽位与路由

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P2
- 前置任务：REP-001
- 主文章类型：cluster
- 研究深度：深度机制
- 任务说明：16384 槽、CRC16、hash tag、MOVED/ASK 重定向、集群总线与 gossip

### CLU-002 集群运维：扩容、reshard 与故障转移

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P2
- 前置任务：CLU-001
- 主文章类型：cluster
- 研究深度：深度机制
- 任务说明：3 主 3 从搭建、在线迁移槽位、节点故障演练、集群限制

## OBS 监控与排障

### OBS-001 INFO 指标与监控体系

- 年度属性：核心
- 权重：S
- 积分：1
- 优先级：P1
- 前置任务：ENV-001
- 主文章类型：observability
- 研究深度：快速沉淀
- 任务说明：INFO 九大区块、关键指标清单、redis-cli --stat、与 PG 监控指标对照

### OBS-002 SLOWLOG 与延迟监控

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P2
- 前置任务：OBS-001
- 主文章类型：observability
- 研究深度：标准研究
- 任务说明：slowlog 配置与解析、LATENCY 事件、延迟来源定位

### OBS-003 大 Key / 热 Key 发现与治理

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P2
- 前置任务：OBS-002
- 主文章类型：observability
- 研究深度：标准研究
- 任务说明：--bigkeys、MEMORY USAGE 扫描、拆分/压缩方案、热 key 应对

### OBS-004 缓存雪崩 / 击穿 / 穿透

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P2
- 前置任务：OBS-003
- 主文章类型：observability
- 研究深度：标准研究
- 任务说明：三种故障模式、TTL 随机化、互斥锁、布隆过滤器、降级方案

## SEC 安全

### SEC-001 ACL 权限体系与安全加固

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P2
- 前置任务：ENV-001
- 主文章类型：security
- 研究深度：标准研究
- 任务说明：ACL 用户/命令/Key 权限、rename-command、TLS、与 PG 角色权限对照

## ENG 工程实践

### ENG-001 缓存架构设计与一致性

- 年度属性：核心
- 权重：L
- 积分：3
- 优先级：P3
- 前置任务：DAT-001
- 主文章类型：engineering
- 研究深度：深度机制
- 任务说明：Cache-Aside 等模式、缓存与数据库一致性、连接池与客户端

### ENG-002 分布式锁与 Redis 原子能力

- 年度属性：核心
- 权重：M
- 积分：2
- 优先级：P3
- 前置任务：DAT-004
- 主文章类型：engineering
- 研究深度：标准研究
- 任务说明：SETNX+Lua 实现、Redlock 争议、与 PG advisory lock 对照

### ENG-003 Redis 7/8 新特性

- 年度属性：弹性
- 权重：S
- 积分：1
- 优先级：P3
- 前置任务：无
- 主文章类型：engineering
- 研究深度：快速沉淀
- 任务说明：Redis Functions、Sharded Pub/Sub、ACL v2、8.x 演进

## OSS 开源生态

### OSS-001 Redis 生态：Valkey 与云托管对比

- 年度属性：弹性
- 权重：S
- 积分：1
- 优先级：P3
- 前置任务：无
- 主文章类型：opensource
- 研究深度：快速沉淀
- 任务说明：Redis vs Valkey 分支、云厂商托管差异（ElastiCache/Tair）、选型建议
