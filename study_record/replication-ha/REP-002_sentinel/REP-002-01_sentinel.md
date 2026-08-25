# REP-002-01 Sentinel 高可用架构

> Redis 8.6.2 / 1 主 2 从 + 3 哨兵（临时实例，端口 6400-6402 / 26379-26381）
>
> REP-001 讲了复制怎么工作；本文讲**自动故障转移**：Sentinel 怎么发现主库挂了、
> 怎么选新主、业务怎么找到新主、旧主回来怎么办。全部本机实测。
> 实测证据：`evidence/sentinel-source.json`、`sentinel-failover-experiment.json`。

---

## 0. 一句话心法

> **Sentinel = 哨兵集群（奇数个），负责"盯主库 → 判死 → 投票选新主 → 通知业务换地址"。**
> 复制（REP-001）负责"数据不丢"，Sentinel 负责"主挂了有人顶上"。
> 对 PG DBA：**这就是 Patroni + etcd 的 Redis 版**——quorum 就是 etcd 的法定人数。

---

## 1. 架构：哨兵自己也要高可用

```text
           ┌──────────────────────────────────────┐
业务客户端 →│ SENTINEL GET-MASTER-ADDR-BY-NAME mymaster │ ← 拿当前主库地址
           └──────────────────────────────────────┘
            26379 ──26380── 26381（3 哨兵，quorum=2，互相 gossip）
                    │
              master 6400 ──→ 6401 / 6402（从库）
```

- Sentinel 之间通过 **gossip**（pub/sub `__sentinel__:hello` 频道）互相发现、交换对主库的看法；
- **quorum=2** 表示"至少 2 个哨兵认为主库挂了"才判客观下线（ODOWN）；
- 哨兵数量取**奇数**（3/5/7），避免投票平局；quorum 建议 = 哨兵数/2 + 1。

---

## 2. 源码机制：SDOWN → ODOWN → 选举 → 切换

Sentinel 每 100ms 跑一次 `sentinelTimer`（sentinel.c），状态机：

```text
sentinelCheckMasterDown（sentinel.c:47 SRI_S_DOWN）
  主库超过 down-after-milliseconds 没响应 PING → 主观下线（SDOWN，本哨兵自己的判断）
       ↓
sentinelCheckObjectivelyDown
  通过 is-master-down-by-addr 问其他哨兵；quorum = 自己(1) + 确认数 >= 配置 quorum
  → 客观下线（ODOWN）→ 触发 failover
       ↓
选举：各哨兵拉票（+vote-for-leader）→ 得票多数当选（+elected-leader，epoch +1）
       ↓
state machine：select-slave → selected-slave → promoted-slave（SLAVEOF NO ONE）
  → reconf-slaves（其余从库改指新主）→ failover-end → switch-master
```

关键参数（默认值）：

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `down-after-milliseconds` | 30000 | 判定主观下线的无响应时长 |
| `failover-timeout` | 180000 | 故障转移超时（含重试） |
| `parallel-syncs` | 1 | 同时向几个从库发起全量重同步（大实例建议 1） |

---

## 3. 实验一：稳态 —— 哨兵认识所有人

```text
SENTINEL MASTER mymaster
  ip=127.0.0.1 port=6400 quorum=2 num-slaves=2 num-other-sentinels=2
SENTINEL REPLICAS mymaster   → 6401 / 6402（flags=slave）
SENTINEL SENTINELS mymaster  → 26380 / 26381（互相发现）
SENTINEL GET-MASTER-ADDR-BY-NAME mymaster → 127.0.0.1 6400
```

- 客户端**不能写死主库地址**：启动时先问哨兵，订阅切换通知，主库变化时换连接；
- 这就是"客户端发现"（对比 PG 的 Patroni REST API / DNS 指向）。

---

## 4. 实验二：kill 主库 → 自动故障转移全流程

配置 `down-after-milliseconds 5000`（加速演示），`shutdown 6400` 后哨兵 leader 日志：

```text
23:17:28.822  # +sdown master mymaster 127.0.0.1 6400          ← 5s 无响应，主观下线
23:17:28.886  # +odown master mymaster 127.0.0.1 6400 #quorum 2/2  ← 客观下线
23:17:28.887  # +new-epoch 1
23:17:28.887  # +try-failover master mymaster
23:17:28.889  # +vote-for-leader 56e16698... 1                  ← 26381 当选
23:17:28.950  # +elected-leader master mymaster
23:17:29.013  # +selected-slave slave 127.0.0.1:6401
23:17:29.976  # +promoted-slave slave 127.0.0.1:6401（SLAVEOF NO ONE）
23:17:31.098  # +failover-end / +switch-master mymaster 127.0.0.1 6400 → 127.0.0.1 6401
```

时间线：**kill → 5s（sdown）→ +2s（odown+选举+提升）→ 完成切换，共 ~8 秒**。
另一个哨兵（26379，未当选）日志显示：

```text
+vote-for-leader 56e16698... 1      ← 投了别人
Next failover delay: I will not start a failover before ...  ← 非 leader 只旁观
```

验证：

```text
GET-MASTER-ADDR-BY-NAME → 127.0.0.1 6401（业务侧发现新主）
6401 role=master，config-epoch 0→1
6402 自动改指 6401（+slave-reconf-done）
SET post-failover:key → 6402 GET 返回 value_ok（数据继续同步）
6401 DBSIZE=50001（50k key 无丢失）
```

---

## 5. 实验三：旧主回归 → 自动降级为从库

```text
重启 6400（旧主，起来时是独立 master）
哨兵日志：+convert-to-slave slave 127.0.0.1:6400 @ mymaster 127.0.0.1 6401
6400 role=slave, master_port=6401, master_link_status=up
GET post-failover:key → value_ok（追平了故障期间的写入）
```

- Sentinel 主动给回归的旧主下发 `REPLICAOF 新主`，**防止脑裂双主**；
- epoch 机制兜底：旧主即使短暂独立，也会因为 config-epoch 旧而让位。

---

## 6. 与 Patroni / PG 对照

| Patroni / PostgreSQL | Redis Sentinel |
| --- | --- |
| etcd/ZooKeeper 存储领导权 + 法定人数 | 哨兵自身 gossip + quorum 投票（无外部依赖） |
| REST API / DCS 发现主库 | `SENTINEL GET-MASTER-ADDR-BY-NAME` |
| 健康检查失败 → 触发选主 | `+sdown/+odown` → 选举 |
| 旧主回归 → 降级为 standby | 旧主回归 → `REPLICAOF` 新主（convert-to-slave） |
| 同步/异步流复制 | 异步复制为主（`WAIT` 可选同步） |
| 数据不丢靠 synchronous_commit 全同步 | 默认可能丢最近 1 秒 + 未同步写入 |

> **Sentinel 不解决丢数据**：故障切换只保证"有主可用"，异步复制下主库已写入但未同步到从库的数据
> 会丢——这是"主从 + 哨兵"方案的固有边界（对比 PG 同步提交可做到 RPO=0）。

---

## 7. DBA 速查

| 场景 | 处置 |
| --- | --- |
| 哨兵自己挂一半 | 哨兵不参与数据，但 quorum 不够会无法 ODOWN；部署奇数且与 Redis 分机 |
| 频繁误切换（网络抖动） | 调大 `down-after-milliseconds`；给哨兵独立监控网络 |
| failover 后延迟飙升 | 大实例 `parallel-syncs` 保持 1，错峰重同步 |
| 客户端拿到旧地址 | 客户端必须用 Sentinel 发现，不能缓存主地址 |
| 新主数据缺失 | 评估 `min-replicas-to-write` + `WAIT` 提升持久性；或换 Cluster |
| 监控 | 盯哨兵日志 `+switch-master`、`INFO sentinel`、`SENTINEL MASTER` 状态 |

---

## 8. 小结

1. **监控**：哨兵每 100ms 探活，超时 → SDOWN（自己的判断）；
2. **判死**：quorum 个哨兵一致 → ODOWN → 触发故障转移；
3. **选举**：+elected-leader（epoch +1）→ 选从库（复制进度最新优先）→ `SLAVEOF NO ONE` 提升；
4. **通知**：`GET-MASTER-ADDR-BY-NAME` + pub/sub 让业务换主；
5. **收敛**：旧主回归自动 `REPLICAOF` 新主，防脑裂；
6. **边界**：异步复制，故障窗口可能丢数据——高可用 ≠ 数据不丢。
