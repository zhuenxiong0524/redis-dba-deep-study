# ENG-003-01 Redis 7/8 新特性：Function、Sharded Pub/Sub、HEXPIRE 与 VSS

> Redis 8.6.2 / 127.0.0.1:6385 实测
>
> 本机基线是 Redis 8.6.2（2026-02 发布），站在 2026 年回头看：Redis 7（2022）引入
> Functions 与 Sharded Pub/Sub，Redis 8.0（2025-04-30）内置 Vector Set、整合 Redis Stack、
> 30+ 性能优化（延迟 -87%、多线程 2x ops），8.4 加 MSETEX 与原子槽迁移 CLUSTER MIGRATION，
> 8.6 持续打磨。本文挑四个"开箱即用"的能力实测：Function、Sharded Pub/Sub、HEXPIRE、VSS。
> 证据：`evidence/features-experiment.json`、`features-source.json`。

---

## 0. 一句话心法

> **新特性解决的是"过去要用外部组件/自己造轮子"的事：Function=服务端脚本管理（替代往
> 客户端塞 EVAL），Sharded Pub/Sub=集群下频道不再全量广播，HEXPIRE=字段级过期（替代
> 拆 key + 定时清理），VSS=内置向量检索（替代自建 HNSW/外挂向量库）。**
> 对 PG DBA：**Redis 8 的形态越来越像 PG 生态"插件进核心"的路线**——过去要装 Redis Stack
> 的东西（向量、Bloom、JSON）逐步内建，和 PG 把很多扩展吸收进 core 是同一条路径。

---

## 1. 四个特性的机制

### 1.1 Redis Function：脚本从"传输"变"注册"

旧做法 EVAL 每次把 Lua 源码发给服务端，ACL 管不到脚本内部、复制/高可用时要保证脚本一致。
Function 把脚本先注册到库里（`FUNCTION LOAD`），调用用 `FCALL 函数名`；实现上
`functions.c:619 fcallCommandGeneric` 从函数库查函数执行，`functions.c:964
functionsCreateWithLibraryCtx` 负责加载库定义。

```text
FUNCTION LOAD "#!lua name=mylib
  redis.register_function('hello', function(keys, args)
    return 'hello ' .. args[1] end)"
FCALL hello 0 redis        → hello redis
```

> 与 PG 对照：FUNCTION LOAD ≈ `CREATE FUNCTION`（一次定义、处处调用、可统一管理/回收），
> EVAL 的散装脚本 ≈ 临时 SQL 片段（到处贴、难治理）。函数是"第一公民"：可被 ACL 授权、
> 可 FUNCTION LIST/KILL 管理，还支持替换（REPLACE）与原子加载。

### 1.2 Sharded Pub/Sub：频道按槽隔离

集群模式下普通 PUBLISH 会把消息广播到每个节点，再转发给所有订阅者——频道消息量大时
全集群放大。Sharded Pub/Sub 用 channel 的 hash slot 决定消息只发到对应分片
（`pubsub.c:718 spublishCommand` → `pubsubPublishMessageAndPropagateToCluster`），
订阅存于 `server.pubsubshard_channels`，与普通频道空间隔离（`ssubscribeCommand`
pubsub.c:726）。

```text
SSUBSCRIBE news            # 客户端 A（分片订阅）
SPUBLISH news sharded-msg  # → A 收到 smessage
PUBLISH news normal-msg    # → A 收不到（普通频道与分片频道完全隔离）
```

> 适用：分布式任务通知、缓存失效广播等"每个分片各收各的"场景；跨分片全局广播仍用普通
> PUBLISH。与 PG 对照：近似 PG logical replication 的"订阅按发布者路由"，只是粒度是槽。

### 1.3 HEXPIRE：字段级过期

hash 里想"某个字段 5 秒后失效"，以前只能拆 key 或自己扫。Redis 7.4 起
`HEXPIRE key seconds FIELDS n field...` 直接给字段设独立 TTL（`t_hash.c:3920
hexpireCommand → hexpireGenericCommand`），并配套 HPEXPIRE/HEXPIRETIME/HTTL 查询：

```text
HSET session:2 token t1 field2 v2
HEXPIRE session:2 5 FIELDS 1 token      → 1（仅 token 字段 5s 过期）
HEXPIRETIME session:2 FIELDS 1 token    → 1787724839
# 6 秒后：HGETALL → field2 v2（token 已消失，field2 保留）
```

> 与 PG 对照：字段级过期 ≈ PG 里"列级数据保鲜"没有直接等价物（PG 通常用分区/定时任务/
> 生成列+清理），Redis 把它做成了命令级原子能力。典型场景：会话里放"token+资料"，token
> 单独过期；购物车商品各自计时的碎片化 TTL。

### 1.4 Vector Set（VSS）：内置向量检索

Redis 8.0 把向量检索做成内置类型（`modules/vector-sets/vset.c:2250 VADD / :2318 VSIM /
:2444 VINFO`），底层 HNSW 近似最近邻（`vset.c:234 createVectorSetObject` → `hnsw_new`），
默认 int8 量化（vset.c:294，VINFO 显示 quant-type=int8），支持降维投影、属性、CAS 更新：

```text
VADD vec VALUES 3 1 0 0 item1          # 新插入 → 1；重复插入 → 0（CAS）
VADD vec VALUES 3 0.997 0.05 0.05 item1a
VADD vec VALUES 3 0.5 0.5 0.5 item3
VSIM vec ELE item1 COUNT 3 WITHSCORES
  → item1=1.0, item1a=0.998747, item3=0.788675   # 近似余弦相似度
VINFO vec → quant-type=int8, hnsw-m=16, vector-dim=3, size=5
```

> 与 PG 对照：PG 向量走扩展（pgvector，HNSW/IVFFlat），Redis 8 把同样的 HNSW 思路内建。
> 选型差别：pgvector 面向"SQL 里 JOIN 检索"，Redis VSS 面向"低延迟 Top-K"；数据量大、
> 要持久化+复杂过滤时 PG+pgvector 更稳，纯在线召回场景 Redis VSS 更顺手。

---

## 2. 8.x 演进时间线（截至 8.6.2）

| 版本 | 时间 | 要点 |
| --- | --- | --- |
| 7.0 | 2022-04 | Functions、Sharded Pub/Sub、ACL v2、AOF 多部分 |
| 7.4 | 2024-07 | HEXPIRE 字段级过期等 |
| 8.0 | 2025-04-30 | Vector Set 内建、整合 Redis Stack、30+ 性能优化（延迟 -87%、多线程 2x ops）、RSALv2/SSPLv1/AGPLv3 三许可 |
| 8.4 | 2025-11 | MSETEX（t_string.c:723）、CLUSTER MIGRATION 原子槽迁移（cluster_asm.c:991） |
| 8.6 | 2026-02 | 本机基线 8.6.2，持续稳定 |

---

## 3. DBA 速查

- Function 治理：`FUNCTION LIST` 看库清单、`FUNCTION KILL` 停执行、ACL 可对 FCALL 授权；
  迁移脚本资产时优先把散落 EVAL 收敛为函数库（一个库一个文件，可版本管理）；
- Sharded Pub/Sub：集群下选 SSUBSCRIBE/SPUBLISH 控制广播放大；普通频道保留给全局广播；
- HEXPIRE：会话/token/限流字段级 TTL 用 `HEXPIRE ... FIELDS`，避免拆 key 爆炸；
  用 `HEXPIRETIME/HTTL` 观测，注意字段级过期在 RDB/AOF 里与 key 级过期同样持久化；
- VSS：`VINFO` 看量化与 HNSW 参数（int8/hnsw-m），`VSIM ... WITHSCORES` 看相似度；
  小数据（<1w 向量）可用 `VSIM` 直查，大数据关注 hnsw-m/降维与内存（量化后 int8 省 4x）；
- 升级关注：8.0 起许可三选一（RSALv2/SSPLv1/AGPLv3），商业托管与再分发前先确认许可条款；
- 监控：`INFO commandstats` 看 FCALL/VADD/VSIM 占比，新特性命令要进客户端白名单与权限设计。

---

## 4. 小结

- Function 把"脚本传输"变"函数注册"，是服务端脚本治理的正解（≈PG 的 CREATE FUNCTION）；
- Sharded Pub/Sub 让集群广播按槽收敛，避免频道放大；
- HEXPIRE 给 hash 字段级 TTL，替代"拆 key + 扫清理"；
- VSS 让向量检索进 core（HNSW+int8 量化），对标 PG+pgvector 的低延迟召回场景；
- 8.x 演进方向明确：把 Redis Stack 能力内建 + 性能/运维能力（原子迁移）持续加强，
  与 PG"扩展进核心"路线同构——DBA 的心智可以平移。
