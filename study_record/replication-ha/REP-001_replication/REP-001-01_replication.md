# REP-001-01 主从复制原理：全量与部分同步

> Redis 8.6.2 / 主 127.0.0.1:6392 ↔ 从 127.0.0.1:6393（临时实例，不碰主库）
>
> 复制是 Redis 高可用的地基（Sentinel/Cluster 都靠它）。本文回答三个问题：
> **第一次怎么同步？断线了怎么续？延迟怎么看？** 全部本机实测。
> 实测证据：`evidence/replication-source.json`、`replication-experiments.json`。

---

## 0. 一句话心法

> **复制 = 主库把自己变成"命令流"（replication stream），从库照着重放。**
> 断线重连后优先"从断点续传"（部分同步，`PSYNC`）；续不上才"推倒重来"（全量，`FULLRESYNC`+RDB）。
> 对 PG DBA 来说：**这就是"流复制 + 复制槽"的 Redis 版**——但 Redis 的 backlog 更像 WAL 保留窗口。

---

## 1. 复制模型与整体流程

```text
从库 replicaof <master> <port>
 └─ 连接主库 → REPLCONF（AUTH/PORT/IP/CAPA 握手）
 └─ PSYNC <replid> <offset>          ← 关键命令（replication.c:1145）
     ├─ offset 在 backlog 内 → +CONTINUE（部分同步，从断点补发）
     └─ 否则 → +FULLRESYNC <replid> <offset>
         └─ 主库 BGSAVE → RDB 直接推给从库（8.x 走独立 rdb-channel）
         └─ 从库加载 RDB → 再续增量命令流
```

状态机（server.h:566-572）：`WAIT_BGSAVE_START → WAIT_BGSAVE_END → SEND_BULK → ONLINE`。
从库侧 `syncWithMaster`（replication.c:3007）负责握手与 RDB 接收。

---

## 2. 实验一：全量同步（第一次连接）

```text
主库：20 万 key（23MB）→ 启动从库（replicaof 127.0.0.1 6392）
主库日志：
  Full resync requested by replica 127.0.0.1:6393 (rdb-channel)
  Starting BGSAVE for SYNC with target: replicas sockets (rdb-channel)
  BGSAVE done, 200000 keys saved, 4089122 bytes written
结果：slave0 state=online | 主从 offset 一致 | 双方 dbsize=200000 | k1/k200000 一致
```

要点：

- 全量 = **BGSAVE 快照 + 增量补发**：BGSAVE 期间的写命令会进 backlog，RDB 发完后从库接着从
  `master_repl_offset` 追增量——所以**从库最终状态和主库一致，不是 RDB 那一刻的快照**；
- 8.x 默认 diskless：RDB 通过独立连接（rdb-channel）直接 socket 传输，不落主库磁盘（PER-001 讲过）；
- 数据量大时全量会占带宽 + 主库 fork（COW 内存放大，PER-001），**第一次全量是容量规划的硬约束**。

---

## 3. 实验二：增量传播与偏移量

```text
主库 SET incr:test hello123
主库 master_repl_offset: 14 → 79
从库 slave_repl_offset:   14 → 79（一致）
从库 GET incr:test → hello123
```

- 每条写命令都会让 `master_repl_offset` 前进，从库追上后 offset 相等；
- **延迟 = master_repl_offset - slave_repl_offset**（以及 `slave0 ... lag=N` 的 ACK 延迟）；
- 对应 PG：`pg_stat_replication` 的 `replay_lsn / sent_lsn` 差。

---

## 4. 实验三：断线重连 → 部分同步（PSYNC 续传）

```text
主库 CLIENT KILL TYPE replica（模拟断线）
主库日志：Partial resynchronization request from 127.0.0.1:6393 accepted.
          Sending 0 bytes of backlog starting from offset 373973
INFO stats：sync_full=1, sync_partial_ok=1, sync_partial_err=0
```

部分同步成立的条件（replication.c:943 `masterTryPartialResynchronization`）：

1. **replid 匹配**：主库 replication ID 没变（没重启、没升过级）；
2. **offset 在 backlog 窗口内**：从库请求的 offset ∈ [backlog 起始, 当前]；
   backlog 默认 **1MB**（`repl-backlog-size`，config.c:3276，最小 16KB）；
3. 满足 → 主库回 `+CONTINUE`，从断点补发——**不重新全量，秒级恢复**。

---

## 5. 实验四：续不上 → 全量重同步

```text
重启从库（replid 未知、offset=-1）→ 必然全量
INFO stats：sync_full 1→2（BGSAVE 253001 keys / 5382825 bytes）
```

- 从库重启后没有复制上下文（replid 为 `?`），主库直接 `FULLRESYNC`；
- **backlog 溢出同理**：断线期间主库写入超过 backlog 大小，旧 offset 被挤出窗口 → 全量；
  所以生产要按"断线窗口 × 写入速率"设置 `repl-backlog-size`，不是越大越好（占内存），
  而是**刚好覆盖预期的断线重连窗口**；
- 从库读旧数据仍可用（master_link_status:down 时 GET 正常），只是"数据可能滞后"。

---

## 6. 实验五：只读、宕机与提升

```text
从库写入       → READONLY You can't write against a read only replica.
主库 shutdown  → 从库 master_link_status:down，GET 照常，写入仍被拒
从库 REPLICAOF NO ONE → role:master，写入恢复 OK
```

- 从库默认 `replica-read-only yes`：**读写分离必须走应用层路由**，从库不会自动拒业务；
- 主库挂掉，从库只是"读得到旧数据"，**不会自动升级**——那是 Sentinel（REP-002）的事；
- `REPLICAOF NO ONE` 手动提升 = 丢失主库宕机后新写入的潜在风险（异步复制语义）。

---

## 7. 与 PG 流复制对照

| PostgreSQL | Redis |
| --- | --- |
| 主库 WAL → 流复制 → 备库重放 | 主库命令流 → 从库重放（本质相同） |
| `primary_conninfo` | `replicaof host port` |
| 复制槽（slot）保 WAL 不被回收 | repl-backlog 保 offset 窗口（无槽概念，超出即全量） |
| `pg_stat_replication`（sent/replay LSN） | `INFO replication`（master/slave offset、lag） |
| 全量 = pg_basebackup | 全量 = RDB 传输 |
| 同步提交 `synchronous_commit` | `WAIT` 命令（少数场景强制等待） |
| 备库可读（hot standby） | 从库可读（replica-read-only） |
| 备库自动提升需 Patroni/etcd | 自动提升需 Sentinel（REP-002） |

---

## 8. DBA 速查

| 场景 | 现象/指标 | 处置 |
| --- | --- | --- |
| 复制延迟 | offset 差越来越大、lag 高 | 查从库负载/带宽；大 key 传播慢（OBS-003） |
| 断线重连 | `sync_partial_ok` 增长 | 正常，无需干预 |
| 频繁全量 | `sync_full` 快速涨 | `repl-backlog-size` 太小，按断线窗口扩容 |
| 从库重启 | 必全量 | 正常；评估大实例重启窗口 |
| 主库 fork 全量 | `latest_fork_usec` 高、延迟抖 | 错峰全量、降低内存写放大（PER-001） |
| 主从数据不一致 | `INFO replication` 对比 offset | 从库只读 + 定期比对抽样 |

---

## 9. 小结

1. **全量同步**：`FULLRESYNC` + BGSAVE/RDB + 增量补发，从库最终一致；
2. **部分同步**：断线重连后 `PSYNC` 命中 backlog → `+CONTINUE` 续传，不重新全量；
3. **续不上就全量**：replid 变了 / offset 超出 backlog 窗口 → `FULLRESYNC`；
4. **backlog 是核心旋钮**：`repl-backlog-size` 按"断线窗口 × 写速率"设置；
5. **从库不自动升级**：只读 + 手动 `REPLICAOF NO ONE`，高可用交给 Sentinel。
