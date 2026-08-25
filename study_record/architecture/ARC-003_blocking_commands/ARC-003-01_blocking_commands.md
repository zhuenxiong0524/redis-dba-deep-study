# ARC-003-01 阻塞命令与延迟问题

> Redis 8.6.2 / 临时实例 127.0.0.1:6394（50 万 key + 30 万元素 List/Set）
>
> ARC-001 证明"一条慢命令拖垮所有人"；本文把**会拖垮服务端的命令**和**只拖垮客户端自己的命令**
> 分清楚，给出实测耗时、阻塞影响与替代方案。
> 实测证据：`evidence/blocking-source.json`、`blocking-experiment.json`。

---

## 0. 一句话心法

> **分两类阻塞：**
> - **服务端阻塞**（危险）：`KEYS` / `SMEMBERS` / `SORT` / 大 `DEL`——占着事件循环，所有客户端陪等；
> - **客户端阻塞**（无害）：`BLPOP` / `BRPOP` / `BLMOVE`——只是请求方挂起，服务端照常干活。
>
> DBA 的活儿：**把第一类从生产里赶出去，把第二类用对**。

---

## 1. 为什么大命令会卡住服务端（源码）

ARC-001 讲过：命令执行在单线程事件循环里，`c->cmd->proc(c)` 跑完才处理下一条。所以：

```text
keysCommand   db.c:1544    遍历整个键空间 O(N)，结果全写进回复缓冲
smembersCommand t_set.c:1558 整个集合序列化 O(N)
sortCommand   sort.c:178   O(N log N) 排序 + 临时内存分配
DEL          db.c:1377    dbSyncDelete 同步释放所有子对象
```

任何一条 O(N) 命令执行期间：不读新连接、不写旧回复、不过期、不刷 AOF——**全局暂停**。

---

## 2. 实测：服务端耗时（50 万 key / 30 万元素）

| 命令 | 服务端耗时 | 客户端墙钟 | SLOWLOG |
| --- | --- | --- | --- |
| `KEYS *` | ~151-170ms | 451ms（含 50 万行回复传输） | 151,699µs |
| `SMEMBERS bset` | ~70ms | 283ms | 69,870µs |
| `SORT blist` | ~198-269ms | 382ms | 268,834µs |
| `LRANGE blist 0 -1` | ~52ms | 226ms | 52,315µs |
| `SCAN 0 COUNT 1000` | **~1.3ms** | 4ms | 不进慢日志 |

> 注意区分：服务端耗时才是"阻塞时长"；客户端墙钟还叠加了回复在网络上的传输时间。
> `SCAN` 每次只取一小批（COUNT），把一次 O(N) 摊成多次小开销——这就是它不阻塞的原因。

---

## 3. 实测：阻塞对其他客户端的影响

```text
KEYS * 执行期间，另一客户端 PING：min=0 avg=2.08 MAX=153ms
SORT   执行期间，另一客户端 PING：min=0 avg=3.97 MAX=263ms
正常基线：max 1-2ms
```

- 50 万 key 的 KEYS 就让 PING 抖动到 153ms；百万级/大 value 场景会是秒级；
- **延迟毛刺与数据规模成正比**：这就是为什么"平时没事，数据涨了突然抖"。

---

## 4. 实测：DEL vs UNLINK（同步 vs 异步释放）

```text
DEL 30 万元素 Set（hashtable）：131ms ← 同步释放，事件循环卡 131ms
UNLINK 30 万元素 Set：           11ms ← 立即返回，bio_lazy_free 后台释放
```

- 释放成本与**元素个数**成正比（30 万 dict 条目逐个 free）；OBS-003 实测 10 万 List 是 DEL 155ms vs UNLINK 39ms；
- **生产删除大 key 一律用 `UNLINK`**；`DEL` 只用于小 key；
- 注意：`UNLINK` 是"快"，但内存真正释放有延迟——`MEMORY USAGE` 不会立刻降。

---

## 5. 实测：BLPOP——阻塞的是客户端，不是服务端

```text
客户端 A：BLPOP q 0（队列空，挂起等待）
1 秒后客户端 B：RPUSH q msg → A 立即返回
A 实际等待：1009ms（客户端阻塞）
等待期间客户端 C 的 PING：min=0 avg=0.14 MAX=1ms（毫无抖动）
```

- BLPOP 等阻塞命令把**客户端**放进等待队列（CMD_BLOCKING，server.h:244），
  服务端事件循环继续跑，数据到达时再唤醒；
- 这类命令是**安全且有用**的（消息队列、限流），不是"阻塞命令"的治理对象。

---

## 6. 治理清单

| 场景 | 危险命令 | 替代方案 |
| --- | --- | --- |
| 枚举 key | `KEYS *` | `SCAN`（分批） |
| 取整个集合 | `SMEMBERS`（大集合） | `SSCAN` 分批 / 业务侧拆 key |
| 全量排序 | `SORT`（大列表） | 排序交给应用 / 用 ZSet 保序 |
| 取整个列表 | `LRANGE 0 -1`（大列表） | 分页 `LRANGE start stop` |
| 删除大 key | `DEL` | `UNLINK` |
| 遍历大 Hash | `HGETALL`（大 hash） | `HSCAN` / 拆分 |
| 危险命令治理 | —— | `rename-command` 或 ACL 禁用（SEC-001） |

配套监控：

- `SLOWLOG`（slowlog-log-slower-than）+ `INFO commandstats` 找"又慢又多"的命令；
- `INFO stats` 的 `instantaneous_*` + LATENCY 事件看毛刺来源；
- 大 key 扫描用 `--bigkeys` / `--memkeys`（OBS-003）提前拆解，避免运行时才发现。

---

## 7. 与 PG 对照

| PostgreSQL | Redis |
| --- | --- |
| 慢 SQL 只拖累自己的会话 | 慢命令拖累所有客户端（全局串行） |
| `pg_stat_statements` 按耗时排序 | `INFO commandstats`（usec/call） |
| 长事务/锁等待拖住 autovacuum | 大 DEL/KEYS 拖住事件循环（含过期/AOF） |
| 大表 `SELECT *` 慢但并发不受影响 | `SMEMBERS/SORT` 慢=全体 PING 抖动 |
| 索引避免全表扫描 | 正确类型+编码避免 O(N) 命令 |

---

## 8. 小结

1. **服务端阻塞**：KEYS/SMEMBERS/SORT/大 DEL 等 O(N) 命令，实测让 PING 抖到 153-263ms；
2. **客户端阻塞**：BLPOP 等挂起的是请求方，服务端零影响（实测 PING max 1ms）；
3. **删除大 key 用 UNLINK**：30 万 Set 实测 131ms → 11ms；
4. **用 SCAN/分页替代全量**：SCAN 单次 1.3ms，不进慢日志；
5. **监控闭环**：SLOWLOG + commandstats 定位，`rename-command`/ACL 禁掉（SEC-001）。
