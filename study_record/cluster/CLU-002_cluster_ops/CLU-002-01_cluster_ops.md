# CLU-002-01 集群运维：扩容、reshard 与故障转移

> Redis 8.6.2 / 本机 3 主 3 从集群（7001-7006，bus=+10000，node-timeout=5000ms）
>
> CLU-001 讲了"一个 key 怎么找节点"（槽位/路由）；本文讲**集群怎么运维**：
> 怎么加从库、怎么在线搬槽、主挂了谁顶上、旧主回来怎么办、多 key 到底有哪些限制。
> 全部本机实测，证据：`evidence/topology-experiment.json`、`reshard-experiment.json`、
> `failover-experiment.json`、`cluster-limits-experiment.json`。

---

## 0. 一句话心法

> **Cluster 运维三件事：加节点（add-node + replicate）、搬槽（reshard/rebalance）、换主（failover）。**
> 核心模型：**数据跟着槽走，槽跟着主人走**——搬槽就是"逐 key MIGRATE + 改槽归属"，
> 换主就是"从库投票升级"。对 PG DBA：**reshard ≈ 在线分区迁移/重新分布，failover ≈ Patroni 选主，**
> 区别是 Redis 把元数据（槽→节点）也做成集群内 gossip，不需要外部 etcd。

---

## 1. 扩容：3 主 → 3 主 3 从

### 1.1 操作

```text
$ redis-cli --cluster add-node 127.0.0.1:7004 127.0.0.1:7001   # 新节点以无槽 master 身份入网（内部 CLUSTER MEET）
$ redis-cli -p 7004 cluster replicate <7001-id>                # 指定从属关系 → 全量同步
```

- 三个从库分别指向 7001/7002/7003，同步完成后 `slave_repl_offset == master_repl_offset`（零积压）；
- 实测坑：连续 add-node 第二个节点时偶发 `[ERR] Nodes don't agree about configuration!`，
  是刚加入的节点还在和全网 gossip 收敛 config epoch，等 1-2 秒重试即成功；
- 新从库做全量同步时主库会 `BGSAVE for SYNC`（fork + COW，同 PER-001 的机制），大实例注意别在高峰期加从库。

### 1.2 从库只读 + 槽跟随

- 从库不拥有槽，但通过集群总线学习槽归属，可用 `READONLY` 命令承接读流量（CLU-001 已述）；
- 主库槽位变化后，从库的**数据**通过复制流跟随，槽归属通过 gossip/configEpoch 跟随。

---

## 2. reshard：在线搬槽

### 2.1 540 槽实测（7002 → 7001）

```text
$ redis-cli --cluster reshard 127.0.0.1:7001 \
    --cluster-from <7002-id> --cluster-to <7001-id> --cluster-slots 540 --cluster-yes
Moving slot 5461 from 127.0.0.1:7002 to 127.0.0.1:7001: ..    # 每个点 = MIGRATE 一个 key
Moving slot 5462 from 127.0.0.1:7002 to 127.0.0.1:7001: ...
```

| 节点 | 迁移前 | 迁移后 | 变化 |
| --- | --- | --- | --- |
| 7001（目标） | 9977 | 10975 | +998 |
| 7002（源） | 10027 | 9029 | -998 |
| 7003 | 9996 | 9996 | 0 |

- **数据跟着槽走**：+998/-998 严格对应，抽查迁移槽内的 key 值正确；
- 从库同步无损：7004=10975、7005=9029、7006=9996，与各自主库一致；
- 每槽的迁移协议 = CLU-001 ASK 实验的手动序列：源 `SETSLOT MIGRATING` → 目标 `SETSLOT IMPORTING`
  → 逐 key `MIGRATE` → 目标/源 `SETSLOT NODE`，`redis-cli --cluster` 工具自动化执行。

### 2.2 在线性验证：迁移中并发读写零失败

后台跑 300 槽 reshard 的同时，前台连续 200 次 `SET`（含落在迁移槽的 key）：

```text
SET OK=200 异常=0
```

- 迁移窗口内客户端会收到 `-MOVED`/`-ASK`/`-TRYAGAIN`，但 `-c` 客户端自动跟随重定向，命令不丢失；
- 迁移期间写入的 key（如 `load:98`）在迁移完成后仍可正确读取；
- **结论：reshard 全程在线，无需停服**——这是对比 PG 在线分区迁移的最大卖点。

### 2.3 rebalance：自动平衡

```text
$ redis-cli --cluster rebalance 127.0.0.1:7001 --cluster-yes
Moving 240 slots from 127.0.0.1:7001 to 127.0.0.1:7005
```

reshard 会人为造成不均衡，rebalance 把槽从最重的节点搬给最轻的节点：
实测后 7005=5462 / 7001=5461 / 7003=5461（总 16384）。

---

## 3. 故障转移：kill 主库全流程

### 3.1 时间线（kill 7002，从库 7005 接管）

| 时刻 | 事件 | 集群状态 |
| --- | --- | --- |
| 13:50:03.77 | `shutdown nosave` 杀掉 7002 | ok |
| 13:50:06~11 | 7005 每秒重连主库失败 | ok |
| 13:50:11.219 | 7003 广播 FAIL（多数主节点弱一致）→ 全集群 **fail** | fail（拒写） |
| 13:50:11.319 | 7005 发起选举，延迟 700ms（rank#0，offset 474662） | fail |
| 13:50:12.042 | 向 7001/7003 请求投票（epoch 10） | fail |
| 13:50:12.049 | **选举获胜，7005 成为新主**（configEpoch=10） | fail |
| 13:50:12.051 | 槽 0-299 + 6001-10922 恢复覆盖 | **ok** |

- **总耗时 ~8.3s，其中真正拒写窗口只有 0.8s**（11.219→12.051）；
- 演练中 +5s/+10s/+15s 的写入全部成功——除非恰好命中那 0.8s 的 CLUSTERDOWN；
- 新主数据完整：7005 DBSIZE=9569（7002 原数据全量接管），故障前写入的 key 全部可读。

### 3.2 选举机制（源码）

```text
clusterHandleSlaveFailover（cluster_legacy.c:4376）
  failover_auth_time = now + 500ms(等 FAIL 传播) + random(0~500) + rank × 1000
  rank = clusterGetSlaveRank()          # 复制偏移量最大者 rank 0，越新越先选
  投票：主节点收到 FAILOVER_AUTH_REQUEST → clusterSendFailoverAuthIfNeeded
  quorum = (cluster->size / 2) + 1      # 多数主节点同意
  限制：每 epoch 只投一票（lastVoteEpoch）；同一主库 node_timeout×2 内不重复投票
```

实测日志与之吻合：`Start of election delayed for 700 ms (rank #0, offset 474662)`
= 500ms 固定 + random(200) + 0×1000。

### 3.3 旧主回归：自动降级为从库

```text
$ redis-cli -p 7002 重启后 info replication
role:slave
master_host:127.0.0.1
master_port:7005        # 自动指向新主！
```

- 旧主 7002 重启后**自动变成 7005 的从库**（configEpoch 10 > 9，旧主认输）；
- 全量同步追平：`slave_repl_offset == master_repl_offset == 474690`，DBSIZE=9569；
- 最终拓扑：7001(540-6000)←7004 / 7005(0-539+6001-10922)←7002 / 7003(10923-16383)←7006。

### 3.4 血的教训：`shutdown nosave` 会丢数据

本次演练埋了个真实事故：CLU-001 的 CLUSTERDOWN 演示对 7003 执行了 `shutdown nosave`，
而该实例 `save ""`（无任何持久化）——重启后 **7003 的 ~8000 个 key 全部丢失**，
集群 state 恢复 ok 但数据没了（后续只补回 2010 个尾部 key，直到重灌基线）。

> **集群高可用只保证"节点挂了有人顶上"，不保证"节点自己重启数据还在"。**
> 集群模式必须配 RDB/AOF（或依赖从库同步后 `CLUSTER FAILOVER`），
> 生产禁止对主库 `shutdown nosave`——这也是从库存在意义的一部分（重启前先让从库顶上）。

---

## 4. 集群限制实测

| 场景 | 结果 |
| --- | --- |
| `MGET` / `MSET` 跨槽 | `CROSSSLOT Keys in request don't hash to the same slot` |
| Lua 脚本（EVAL）跨槽 key | `CROSSSLOT`（脚本内多 key 必须同槽） |
| MULTI 跨槽（非重定向连接） | 入队时即报错 → `EXECABORT Transaction discarded because of previous errors` |
| MULTI 同槽 hash tag（直连槽主） | 正常执行 |
| **带 `-c` 重定向客户端中途 MOVED** | **redis-cli 跟随重定向换连接 → `EXEC without MULTI`，事务上下文被破坏** |
| 无 key 命令（PING / CLUSTER KEYSLOT） | 任意节点可用 |

要点：
- **跨节点原子性只能靠 hash tag 钉同槽**；事务/Lua/多 key 命令必须在设计期保证同槽；
- 重定向客户端会静默破坏 MULTI 上下文——客户端库必须在事务前把连接钉在槽主节点上
  （很多库的 `execute_transaction` 就是这么做的）。

---

## 5. 与 PG / 分布式运维对照

| Redis Cluster 运维 | PostgreSQL / 分布式类比 |
| --- | --- |
| `add-node` + `CLUSTER REPLICATE` | 加 standby / 新节点重分布 |
| `reshard` 逐槽 MIGRATE | 在线分区迁移 / 逻辑复制搬数据 |
| `rebalance` | 分区再平衡（如 Citus rebalance） |
| 从库投票 failover（8s 级） | Patroni + etcd 选主（秒级） |
| configEpoch 仲裁旧主降级 | Patroni 的 leader epoch |
| 槽缺失 → 全集群拒写 | 分区未覆盖 → 只读/拒写策略 |
| hash tag 限同槽 | 分区键选择约束 |

---

## 6. DBA 速查

- 扩从库：`add-node` 后 `cluster replicate <master-id>`，避开高峰（主库 fork 全量同步）；
- 搬槽：`redis-cli --cluster reshard <node> --cluster-from <id> --cluster-to <id> --cluster-slots N --cluster-yes`；
- 平衡：`redis-cli --cluster rebalance <node> --cluster-yes`（按槽数自动搬）；
- 故障判断：`CLUSTER INFO` 看 `cluster_state`/`cluster_slots_pfail|fail`；`CLUSTER NODES` 看 `fail` 标记；
- 故障转移耗时 = node_timeout(PFAIL 判定) + FAIL 传播 + 选举延迟(500+rand+rank×1000) + 切换；
- 主库重启：先确认从库已接管，旧主起来自动降级；有持久化再考虑 `CLUSTER FAILOVER` 优雅切回；
- 多 key 设计：事务/Lua 必须同槽（hash tag），跨槽只能放弃原子性或用应用层补偿；
- 从库读：`READONLY` + 客户端路由到从库，注意复制延迟。

---

## 7. 小结

- 扩容 = `add-node`（入网）+ `cluster replicate`（全量同步），加从库是集群最便宜的高可用手段；
- 搬槽 = MIGRATE 逐 key + SETSLOT 状态机，**在线、无损、可回滚**，rebalance 自动均衡；
- 故障转移 = PFAIL→FAIL（多数弱一致）→ 从库投票（offset 最新者优先）→ configEpoch 接管，
  实测 8.3s 完成、0.8s 拒写窗口、数据零丢失、旧主回归自动降级；
- 限制 = 多 key 原子性锁死在单槽，hash tag 是唯一出路，重定向客户端会破坏 MULTI 上下文；
- 最重要的教训：**高可用 ≠ 数据安全**，集群主库照样要持久化，`shutdown nosave` 是事故触发器。
- 下一步：SEC-001 ACL 权限体系，或 ENG-002 分布式锁（SETNX+Lua 与 advisory lock 对照）。
