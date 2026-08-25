# 过期、TTL 与键删除机制

> Redis 8.6.2 / Debian 12 (Linux 5.10.0-38-amd64) / x86_64 / 实例 127.0.0.1:6379

缓存的最大特征就是"会过期"。本任务研究 Redis 的过期模型：TTL 命令语义、过期时间如何存储
（Redis 8 kvobj 元数据）、以及**两种删除路径**——惰性删除（`expireIfNeeded`）与定期删除
（`activeExpireCycle`），并用 `expired_keys` 统计做实验验证，最后与 PG 的 vacuum 心智对照。

---

## 1. TTL 命令语义（实测）

| 命令 | 返回值 | 说明 |
| --- | --- | --- |
| `TTL key` | `-2` | key 不存在 |
| `TTL key` | `-1` | key 存在但无过期 |
| `TTL key` | `N` | 剩余秒数，**四舍五入**（实测剩 1291ms → 1） |
| `PTTL key` | 毫秒级剩余 | 精确到 ms |
| `EXPIRETIME` / `PEXPIRETIME` | 绝对 unix 时间 | Redis 7+，适合对账 |
| `PERSIST key` | 1/0 | 移除过期，成功返回 1 |

`EXPIRE` 还支持条件旗标（Redis 7+）：

```text
EXPIRE key 200 NX   # 仅当 key 当前无过期时设置（实测：已有过期→0，无过期→1 并生效）
EXPIRE key 300 GT   # 新过期时间必须大于当前（实测 →1，TTL 变为 300）
XX / LT 同理        # 仅当已有过期 / 新值必须小于当前
```

> 与 PG 对照：Redis 的 TTL 是 **key 级** 的；PG 是**行级**（如 `pg_ttl` 类扩展或业务字段）——
> Redis 把"过期"下沉到存储引擎，访问路径天然感知，无需应用层过滤。

---

## 2. 过期时间存哪：Redis 8 的 kvobj 元数据

Redis 8 的 kvobj 把 key 与元数据（含过期时间）内联在同一对象：

- 过期时间占用**一个 8 字节元数据块**（`KEY_META_MASK_EXPIRE`，class 0），放在 kvobj 之前；
- 实测内存：同一 key/value，无过期 56B → 带 `EX` 64B，**+8B**（见 evidence/expire-metadata-memory.json）；
- 同时维护 `db->expires` 字典，供主动删除周期**采样**（不用扫全表，是随机抽查）。

---

## 3. 惰性删除：访问时删除（expireIfNeeded）

### 3.1 源码路径

```text
GET/SET 等任何命令 → lookupKeyRead/Write → expireIfNeeded (db.c L2905)
    → keyIsExpired: now > expire?（db.c L2847）
    → 已过期 → deleteExpiredKeyAndPropagate → 返回 KEY_DELETED
    → 调用方按"key 不存在"处理
```

### 3.2 实验

```text
SET dat003:lazy:key2 value2
PEXPIRE dat003:lazy:key2 100
sleep 300ms
GET dat003:lazy:key2   → nil     # 访问瞬间惰性删除
EXISTS dat003:lazy:key2 → 0
```

关键点：

- **惰性删除只发生在"被访问"的 key 上**；无人访问的过期 key 会一直占内存，直到主动周期来收；
- 从库（replica）**不会本地惰性删除**（源码 `masterhost != NULL` 分支直接返回 KEY_EXPIRED）：
  过期由主库统一判定并合成 DEL 广播，避免主从时钟偏差导致数据不一致；
- 统计计数器：`expired_keys`（所有删除），`expired_keys_active`（其中由主动周期删除的）。

---

## 4. 定期删除：主动回收（activeExpireCycle）

### 4.1 两档触发（源码 expire.c L287）

| 档位 | 调用点 | 频率/预算 | 行为 |
| --- | --- | --- | --- |
| SLOW | `serverCron → databasesCron` | 每 100ms（hz=10）一次，最多占用周期 **25% ≈ 25ms** | 主要回收通道 |
| FAST | `beforeSleep`（每次事件循环） | 最长 **1000µs**，且与上次 FAST 至少间隔其耗时 | 借事件循环空闲快速补收 |

算法要点（DBA 需要知道的"为什么它不会卡死 Redis"）：

- **随机采样**：从 `db->expires` 抽查，每 DB 每轮最多 **20 个 key**（`ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP`），
  **不是全表扫描**——这是与 PG vacuum 最大的不同；
- **自适应**：采样过期率 < 10%（`ACCEPTABLE_STALE`）就提前结束；过高则下轮更积极；
- **时间上限**：SLOW/FAST 都有硬性耗时上限，超限立即让出 CPU，**不会阻塞主线程业务**；
- `active-expire-effort 1~10` 可调（默认 1），内存压力大时可加大。

### 4.2 实验：5000 个短 TTL key，全程不访问

```text
创建：SET dat003:bulk:1..5000 + PEXPIRE 400ms   → DBSIZE=5000
3 秒后（未访问任何 key）：DBSIZE=1
INFO stats：expired_keys 4 → 5003（+4999）
            expired_keys_active 4 → 5003（+4999）   # 全部由主动周期删除
```

这个实验直接证明：**即使没有任何读请求，过期 key 也会被后台周期回收**，
且 `expired_keys_active` 与 `expired_keys` 增长完全一致（无访问 = 无惰性路径）。

---

## 5. 与 PostgreSQL vacuum 心智对照

| PostgreSQL | Redis | 差异 |
| --- | --- | --- |
| autovacuum 回收死元组 | `activeExpireCycle` 回收过期 key | PG 是**全表/分页扫描**，Redis 是**随机采样**，成本可控 |
| 死元组可见性（MVCC） | 过期 key 逻辑上不可见 | Redis 无版本链，删除即物理回收 |
| vacuum 不及时 → 表膨胀 | 主动周期跟不上 → 过期 key 占内存 | 都是"回收滞后"，Redis 有 `expired_stale_perc` 指标 |
| `pg_stat_user_tables.n_dead_tup` | `INFO stats.expired_keys` / `expired_stale_perc` | 监控口径不同 |
| 手动 `VACUUM` | 无手动命令 | Redis 靠惰性 + 周期两条路，天然自动 |
| 从库 vacuum 本地做 | 从库过期由主库 DEL 驱动 | Redis 避免时钟偏差，主从一致性强于"各删各的" |

**一句话**：PG 的回收是"扫描型"（重、可延迟），Redis 的回收是"采样型"（轻、有预算上限）+
"访问即删"（快、确定性）。Redis 把"清理过期"做成了 O(1)~O(小样本) 的事，代价是过期 key
在无人访问时可能短暂滞留内存——这正是 `maxmemory` + 淘汰策略要兜底的原因。

---

## 6. 运维速记

1. 排查"内存为什么没释放"：先看 `INFO stats` 的 `expired_keys` 增量与 `expired_stale_perc`；
2. 大批量设短 TTL（如清缓存）会瞬间抬高主动周期负载，但 SLOW/FAST 有硬预算，不必过度担心；
3. 从库的过期不是本地时钟判定，主库挂了从库提升时，过期 key 可能在接管窗口内"多活一会"；
4. TTL 秒级显示是四舍五入，精确排障用 `PTTL`/`PEXPIRETIME`；
5. 每个 key 的过期元数据 +8B（Redis 8 kvobj），海量短生命周期 key 时这也是内存成本。

## 7. 路径与证据

```text
任务目录  study_record/datatype/DAT-003_expire_ttl_delete/
证据      evidence/ttl-command-semantics.json   (TTL/EXPIRE NX/GT/PERSIST)
          evidence/lazy-delete.json            (GET 触发惰性删除)
          evidence/active-expire.json          (5000 key 主动回收 + 统计)
          evidence/expire-metadata-memory.json (kvobj 过期 +8B)
          evidence/expire-source.json          (源码路径取证)
源码      /data/redis_pkg/redis-stable/src/expire.c / db.c / server.c
```
