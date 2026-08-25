# PER-002-01 AOF 与混合持久化

> Redis 8.6.2 / 127.0.0.1:6379
>
> PER-001 讲了 RDB 快照；本文讲 **AOF（命令日志）+ 8.x 混合持久化**，并回答 PG DBA 最关心的问题：
> **RDB+AOF 是不是就是 PG 的"全量数据 + WAL"？** ——是。
> 实测证据：`evidence/aof-multi-part.json`、`aof-crash-recovery.json`。

---

## 0. 一句话心法

> **RDB 是"照相机"（瞬间定格），AOF 是"录像机"（连续记录）；两者组合 = PG 的"全量数据 + WAL"。**
> RDB 管恢复快，AOF 管丢得少，8.x 默认同时启用（混合持久化）。

---

## 1. AOF 是什么

**AOF（Append Only File）= 命令日志**：把每一条写命令追加到文件，崩溃后重放命令恢复数据。

实测文件内容（`strings` 看 incr.aof）：

```text
SELECT / SET auto:key:1 / SET auto:key:2 ...   ← 就是命令流，回放=重新执行
```

对比 RDB：RDB 是二进制快照（最终状态），AOF 是文本命令流（操作过程）。所以：
- RDB 加载 = 直接反序列化（快）；
- AOF 回放 = 重新执行每条命令（慢，但**丢得少**）。

---

## 2. appendfsync：三种刷盘策略（数据安全与性能的取舍）

| 策略 | fsync 时机 | 崩溃丢失 | 性能 | 适用 |
|---|---|---|---|---|
| `always` | 每条命令 | 0（一条不丢） | 最慢 | 强一致场景（很少用） |
| `everysec` | 每秒批量 | **最多 1 秒** | 快（本机默认） | 生产默认，性价比最高 |
| `no` | 交给 OS | 可能数秒~数十秒 | 最快 | 可容忍较大丢失 |

> 与 PG 对照：PG `synchronous_commit=on` 每次提交即 fsync WAL（≈ always 但更重）；
> Redis 默认 `everysec` 是"缓存角色"的取舍——丢了能重建，1 秒窗口可接受。

---

## 3. 8.x 多部件 AOF：base + incr + manifest

8.x 的 AOF 不再是单个大文件，而是目录 `appendonlydir/` 里的三部分（实测）：

```text
appendonly.aof.4.base.rdb   # base：最近一次重写的全量快照（RDB 格式，1035B）
appendonly.aof.4.incr.aof   # incr：base 之后追加的写命令（488B）
appendonly.aof.manifest     # 清单：当前用哪份 base + 增量从哪个偏移开始
```

manifest 实测内容：

```text
file appendonly.aof.4.base.rdb seq 4 type b      ← base 文件
file appendonly.aof.4.incr.aof seq 4 type i startoffset 2306507  ← 增量及起始偏移
```

**加载流程**（崩溃/重启）：读 manifest → 加载 base RDB（全量）→ 从 startoffset 回放 incr AOF（增量）→ 就绪。

---

## 4. AOF 重写（rewrite）：日志压缩

- **触发**：`auto-aof-rewrite-percentage 100` + `auto-aof-rewrite-min-size 32MB`——文件增长翻倍且 ≥32MB 自动重写（或手动 `BGREWRITEAOF`）；
- **机制**：fork 子进程生成**新 base RDB**（当前最终状态）+ 记录期间增量 → manifest 原子切换到新序列（seq 递增）；
- **实测**：`aof_current_size` 28MB → 1KB（只保留最终状态，历史命令全部丢弃）；
- 为什么需要：AOF 一直追加会无限膨胀（比如同一 key 改 100 次就记 100 条），重写后只留最新状态。

> 对照 PG：AOF rewrite ≈ WAL 归档 + checkpoint 推进（旧日志回收）；manifest 切换 ≈ `pg_control` 更新检查点。

---

## 5. 崩溃恢复实测（kill -9）

```bash
# 临时实例写 4 个 key → 直接 kill -9（模拟断电/崩溃，不优雅关闭）→ 重启
```

实测结果：**4/4 key 全部恢复**（everysec 下本次无丢失），恢复日志：

```text
DB loaded from base file appendonly.aof.1.base.rdb: 0.001 seconds   ← 全量
DB loaded from incr file appendonly.aof.1.incr.aof: 0.000 seconds   ← 增量
DB loaded from append only file: 0.001 seconds
Ready to accept connections tcp                                     ← 恢复完成才对外服务
```

> 注意：恢复日志出现在"Ready to accept connections"之前——**加载期间不服务**（结合 ENV-002 §1.1.1：启动=全量加载）。

---

## 6. RDB + AOF = PG 的"全量数据 + WAL"

| PG | Redis | 说明 |
|---|---|---|
| 数据文件（堆表/索引） | **base RDB** | 全量基线 |
| **WAL**（预写日志） | **incr AOF** | 快照后的增量 |
| checkpoint | BGSAVE / AOF rewrite | 推进恢复基线 |
| `pg_control` | `appendonly.aof.manifest` | 记录当前 base + 偏移 |
| 崩溃恢复：checkpoint + WAL | 启动：base + incr 回放 | 同一思路 |

**三个关键差异（PG DBA 重点）**：

1. **日志性质**：PG WAL 是**物理日志**（页级字节），AOF 是**命令日志**（`SET key val`，回放=重执行）——AOF 更像 MySQL binlog（逻辑日志）；
2. **fsync 强度**：PG `synchronous_commit=on` 每次提交 fsync（零丢失）；Redis `everysec` 每秒 fsync（**最多丢 1 秒**）；
3. **恢复范围**：PG 崩溃恢复在磁盘上进行（数据已在盘）；Redis 是**全量重载进内存**（启动即恢复，预热成本大）。

---

## 7. 小结

- AOF = 命令日志，`appendfsync` 三档：`always`（0 丢失）/ `everysec`（≤1s，默认）/ `no`（丢更多）；
- 8.x 多部件：base RDB + incr AOF + manifest，重写=生成新 base + 原子切换，日志不无限膨胀；
- 崩溃恢复：kill -9 实测 4/4 恢复，流程 = manifest → base → incr 回放 → 就绪；
- RDB+AOF 组合就是 PG 全量+WAL 的 Redis 版，差异在日志性质（命令 vs 物理）、fsync 强度、恢复范围（内存重载）。

**后续深化**：备份策略与恢复演练（redis-check-rdb/aof、点表恢复、与 pg_basebackup 对照）进入 **PER-003**。
