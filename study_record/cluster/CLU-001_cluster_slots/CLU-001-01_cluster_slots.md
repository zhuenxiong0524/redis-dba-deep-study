# CLU-001-01 Cluster 分片：槽位与路由

> Redis 8.6.2 / 本机 3 主集群（7001=0-5460、7002=5461-10922、7003=10923-16383，bus=+10000）
>
> REP 系列解决"单点数据怎么冗余、主挂了怎么顶上"；本文进入**多节点数据分片**：
> 一个 key 到底落在哪个节点？客户端怎么知道去问谁？槽在迁移时又是怎么协调的？
> 全部本机实测 + 源码取证（`evidence/cluster-source.json`、`routing-experiment.json`、
> `distribution-experiment.json`、`gossip-bus-experiment.json`、`clusterdown-experiment.json`）。

---

## 0. 一句话心法

> **Cluster = 16384 个槽做"虚拟分片"：CRC16(key) 低 14 位定槽，槽 → 节点映射做成一张全局路由表。**
> 客户端不是傻傻连一台，而是**先问、被重定向（MOVED/ASK）再换节点**；节点之间靠
> **集群总线（独立端口）gossip** 交换"谁活着、谁挂了、槽归谁"。
> 对 PG DBA：**槽 ≈ 分区表的分区键哈希，路由表 ≈ 全局元数据，gossip ≈ 节点间的心跳+告警扩散。**

---

## 1. 架构：三种角色各司其职

```text
                          ┌─────────────────────────────┐
   redis-cli -c           │       集群总线（+10000 端口）      │
  ┌──────────┐  MOVED/ASK │  7001@17001 ── 7002@17002     │
  │ 客户端路由  │──────────→│      └────── 7003@17003       │
  └──────────┘   重定向    │  PING/PONG(每秒) + gossip 扩散  │
   client port 7001~7003  └─────────────────────────────┘
```

- **槽（slot）**：`0..16383` 共 16384 个，是分片的最小单元；每个槽恰好属于一个主节点；
- **路由表**：`clusterState.slots[16384]`（cluster_legacy.h:335）槽 → 节点指针；
  另有 `migrating_slots_to[]` / `importing_slots_from[]` 两张迁移状态表；
- **集群总线**：每节点额外监听 `客户端端口 + 10000`，只跑 gossip 协议，与数据端口隔离；
- **分片粒度是"槽"不是"key"**：一个节点服务一段槽区间，key 靠哈希落入区间。

---

## 2. 源码机制

### 2.1 槽位计算：CRC16 低 14 位 + hash tag

`keyHashSlot`（cluster.h:59，static inline）：

```text
1. 找 key 中第一个 '{'
2. 没有 '{'            → slot = crc16(整个 key) & 0x3FFF
3. 有 '{'，找其后第一个 '}'
   ├─ 没有 '}' 或 {} 中间为空 → 仍哈希整个 key
   └─ 有内容            → 只哈希 { 和 } 之间的子串
```

- `crc16` 是 XMODEM 变体（crc16.c：poly=0x1021，init=0，表驱动）；`& 0x3FFF` = 取低 14 位；
- **hash tag 只认第一对 `{...}`**：`{user1000}.following` 和 `{user1000}.followers` 必然同槽，
  所以多 key 操作（MSET/MGET/事务/Lua）能用 hash tag 约束到同槽执行；
- 边界：`foo{{bar}}zap`（第一对 {} 中间为空）→ 哈希整 key；`foo{bar}{zap}` → 只看第一对 `{bar}`。

### 2.2 路由判定：getNodeByQuery 的统一路径

`getNodeByQuery`（cluster.c:1191）把单命令和 MULTI/EXEC 统一成一条代码路径：

```text
所有 key 逐个算槽
 ├─ 首个 key 定下 slot 和目标节点 n（getNodeBySlot）
 ├─ 后续 key 槽不一致                    → CROSSSLOT
 ├─ 槽无人服务                            → CLUSTERDOWN（DOWN_UNBOUND）
 ├─ 本节点是槽主人，但槽在 MIGRATING：
 │    ├─ key 全不在（missing && !existing）→ ASK 到迁移目标
 │    └─ 部分在部分不在                   → TRYAGAIN
 ├─ 本节点在 IMPORTING 且客户端带 ASKING   → 本节点直接服务
 └─ 否则 n != 自己                        → MOVED 到 n
```

`clusterRedirectClient`（cluster.c:1443）按错误码回不同报文，MOVED/ASK 格式：

```text
-MOVED <slot> <ip>:<port>      （客户端该换节点，路由表已变，永久重定向）
-ASK   <slot> <ip>:<port>      （槽在迁移中，先 ASKING 再发原命令，一次性放行）
-CROSSSLOT ...                 （多 key 不同槽）
-CLUSTERDOWN ...               （集群整体不可用）
-TRYAGAIN ...                  （迁移中多 key 操作不稳定）
```

> **MOVED 与 ASK 的本质区别**：MOVED 是"路由表改了，以后都去新节点"；
> ASK 是"槽正在搬，目标节点还没正式接管，先放行这一次"（目标节点不更新自己的槽表）。

### 2.3 集群总线与 gossip

- 报文头 `clusterMsg`（cluster_legacy.h:220）：魔数 **"RCmb"**、总长、协议版本 1、类型、
  count（gossip 条目数）、epoch；gossip 条目 `clusterMsgDataGossip` 带节点名/ip/port/cport/标志位；
- `clusterCron`（cluster_legacy.c:4731）每 100ms 跑一次：**每 10 次迭代向随机节点发 PING**
  （约每秒 1 次），并对 pong 超时超过 node_timeout/2 的节点主动补 PING；
- 每条 PING/PONG 携带 `nodes/10`（至少 3）条 gossip；**PFAIL 节点全部优先携带**，加速故障扩散；
- 收到 gossip 的 PFAIL/FAIL 标记且发送者是 master → 记入 `fail_reports`；
  `markNodeAsFailingIfNeeded` 里 **quorum = 主节点数/2 + 1**（多数弱一致）才升 FAIL；
- FAIL 后通过 CLUSTERMSG_TYPE_FAIL 广播强制全网标记。

---

## 3. 实验一：槽位计算与分布

### 3.1 CLUSTER KEYSLOT 实测 vs 独立实现

用源码 crc16 表 + keyHashSlot 逻辑写独立 C 程序（`evidence/slot_distribution.c`），
11 个用例与 `redis-cli CLUSTER KEYSLOT` 输出**逐字一致**：

| key | slot | 说明 |
| --- | --- | --- |
| `foo` | 12182 | 整 key 哈希 |
| `user1000` | 3443 | 整 key 哈希 |
| `{user1000}.following` | 3443 | 只哈希 user1000 |
| `{user1000}.followers` | 3443 | 与上同槽（hash tag 效果） |
| `foo{{bar}}zap` | 4015 | 第一对 {} 为空 → 整 key |
| `foo{bar}{zap}` | 5061 | 只看第一对 {bar} |

### 3.2 10 万随机 key 分布

| 指标 | 值 |
| --- | --- |
| 覆盖槽位数 | 16355 / 16384 |
| 每槽平均 | 6.1 |
| 每槽最多 | 16 |
| 极差 | 16 |

结论：CRC16 低 14 位**分布均匀、无热点槽**；但 hash tag 会人为制造单槽热点——
1000 个 `{hotuser}.*` key 全部命中 1 个槽（对应 PG 分区键倾斜的坑）。

### 3.3 30k 真实写入分布

`redis-cli -c` 灌入 30000 个 `load:N` key 后三节点 DBSIZE：

| 节点 | DBSIZE |
| --- | --- |
| 7001 | 9978 |
| 7002 | 10027 |
| 7003 | 9996 |

**运营坑**：`redis-cli --pipe -c` 不跟随 MOVED（30000 条报 errors: 20023，跨槽全失败）；
只有逐行 `-c` 模式会自动重定向。批量灌数据务必用客户端库的集群模式或逐行 -c。

---

## 4. 实验二：MOVED / CROSSSLOT / ASK 重定向

### 4.1 MOVED：客户端不处理就丢命令

```text
$ redis-cli -p 7001 SET foo v1        # foo→slot 12182→7003
MOVED 12182 127.0.0.1:7003            # 命令未执行！

$ redis-cli -c -p 7001 SET foo v1     # -c 自动跟随重定向
OK
$ redis-cli -p 7003 GET foo           # 数据确实在 7003
v1
```

### 4.2 CROSSSLOT：多 key 必须同槽

```text
$ redis-cli -p 7001 MSET foo x bar y
CROSSSLOT Keys in request don't hash to the same slot

$ redis-cli -c -p 7001 MSET {tag}a 1 {tag}b 2   # hash tag 同槽 → OK
OK
```

### 4.3 ASK：迁移中的一次性放行（testkey，slot 4757）

| 步骤 | 结果 |
| --- | --- |
| 7001 `SETSLOT 4757 MIGRATING <7002-id>` | OK |
| 7002 `SETSLOT 4757 IMPORTING <7001-id>` | OK |
| 7001 `MIGRATE 127.0.0.1 7002 testkey 0 5000` | OK（key 到 7002） |
| 7001 `GET testkey` | `ASK 4757 127.0.0.1:7002` |
| 7002 `GET testkey`（未 ASKING） | `MOVED 4757 127.0.0.1:7001`（导入态不认普通请求） |
| 7002 同连接 `ASKING` + `GET testkey` | `v1`（放行） |

> **ASKING 是连接级一次性标志**：跨连接无效。上面用两条独立 redis-cli 复现过——
> 第二条连接没带 ASKING，照样 MOVED。这也是很多客户端库迁移期偶发
> `-MOVED`/`-ASK` 处理不一致的根因之一。

---

## 5. 实验三：集群总线与 gossip

- bus 监听：`7001@17001`（客户端端口 +10000），报文魔数 `RCmb`；
- 6 秒窗口 gossip 计数（`CLUSTER INFO`）：

| 指标 | t0 | t+6s | 增量 |
| --- | --- | --- | --- |
| ping_sent | 23 | 31 | +8 |
| pong_sent | 21 | 28 | +7 |
| ping_received | 19 | 26 | +7 |
| pong_received | 23 | 31 | +8 |

→ 与源码注释一致：**约每秒 1 次 PING 到随机节点**，PONG 对称回复。

- 故障扩散可观测：kill 7003 后 `cluster_stats_messages_fail_sent:1 / fail_received:1`
  （FAIL 消息全网广播）；
- 8.x 的 `CLUSTER INFO` 还新增 `cluster_slot_migration_active_tasks` / `trim_running`
  等字段（Redis 8 的槽迁移后台任务，CLU-002 会实测）。

---

## 6. 实验四：CLUSTERDOWN —— 槽不全覆盖 = 全集群拒写

`require-full-coverage` 默认开启：**只要有一个槽没有健康节点服务，整个集群拒绝写入。**

| 时间 | 事件 | cluster_state | 写入 |
| --- | --- | --- | --- |
| +0s | shutdown 7003 | ok（slots_ok=16384） | 正常 MOVED |
| +7s | 7003 失联 → 5461 槽 PFAIL | ok（slots_pfail=5461） | 正常 MOVED |
| +15s | PFAIL→FAIL（多数弱一致） | **fail**（slots_fail=5461） | `CLUSTERDOWN The cluster is down` |
| 重启 7003 后 | FAIL 清除、gossip 恢复 | ok（slots_ok=16384） | 恢复 |

要点：PFAIL 只是"我觉得它可能挂了"，升 FAIL 需要多数主节点一致（quorum=size/2+1）；
FAIL 之前集群仍对外服务（哪怕部分槽已不可用），FAIL 之后整体拒写。

---

## 7. 与 PG / 分布式概念对照

| Redis Cluster | PostgreSQL / 分布式类比 | 说明 |
| --- | --- | --- |
| 16384 槽 = CRC16(key) 低 14 位 | 分区键哈希分区 | 分片单元是槽，不是 key |
| hash tag `{user}` | 复合分区键/关联列 | 同槽多 key 操作的唯一手段 |
| MOVED（永久） / ASK（一次性） | 无直接对应 | 更像 DNS/TCP 重定向语义 |
| CROSSSLOT | 跨分区/跨库查询 | 多 key 原子性被限制在单槽 |
| 集群总线 gossip + PFAIL/FAIL | etcd/Patroni 心跳与法定人数 | 弱一致多数判定，无 Paxos 强一致 |
| require-full-coverage=yes 全集群拒写 | 分布式事务"全有或全无" | 可用性/一致性取舍（可用性优先可设 no） |
| configEpoch 解决槽归属冲突 | 分区元数据版本/epoch | 新纪元压旧纪元，防分脑 |

---

## 8. DBA 速查

- 看路由：`CLUSTER SLOTS` / `CLUSTER NODES`（列：id、ip:port@bus、flags、master、
  ping/pong 时间、config-epoch、link 状态、槽区间）；
- 算槽：`CLUSTER KEYSLOT <key>`；生产先算再决定集群内联机；
- 客户端要求：**必须支持集群模式（跟随 MOVED/ASK）**；普通连接不做重定向，写错节点直接报错；
- 多 key 设计：需要原子多 key → 用 hash tag 钉同槽；同槽 key 太集中 → 槽热点/单节点压力；
- 迁移期：客户端会看到 `-ASK`/`-TRYAGAIN`，属正常；`--pipe` 类工具不走重定向，避开；
- 排障：`cluster_state:fail` 先看 `cluster_slots_fail`/`pfail` 找失联槽，再查该槽节点与集群总线连通性。

---

## 9. 小结

- 分片 = `CRC16(key) & 0x3FFF`，hash tag 只认第一对 `{}`，可精确预测 key 归属；
- 路由 = `getNodeByQuery` 一张判定表：CROSSSLOT / MOVED / ASK / TRYAGAIN / CLUSTERDOWN；
- 协调 = 集群总线（+10000 端口）每秒 gossip，PFAIL→FAIL 需多数主节点弱一致；
- 实测确认：分布均匀（30k key 三分天下）、重定向报文与源码一致、槽位缺失整集群拒写；
- 下一步 CLU-002：3 主 3 从、`redis-cli --cluster reshard` 在线迁移槽位、主节点故障自动提升
  ——把本文的 MIGRATING/IMPORTING/ASK 在真实迁移流程里再看一遍。
