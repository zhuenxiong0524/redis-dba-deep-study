# PER-001-01 RDB 快照机制与触发时机

> Redis 8.6.2 / 127.0.0.1:6379（2026-08-25 环境重建后实例）
>
> 回答"**啥时候会落盘？落盘前发生了什么？**"。本文聚焦 RDB（AOF 详细机制见 PER-002）。
> 所有数值均本机实测（证据：`evidence/persistence-triggers.json`）。

---

## 0. 一句话心法

> **RDB 是"快照"，AOF 是"流水账"；触发靠条件（save / fsync），落盘靠 fork + 子进程。**
> 记住三个计数器就懂了：`rdb_changes_since_last_save`（距上次快照的变更数）、`aof_current_size`（AOF 大小）、`latest_fork_usec`（fork 耗时）。

---

## 1. RDB 触发时机（4 种）

### 1.1 自动 save 条件（本机实测配置）

```text
save 3600 1        # 3600 秒内 ≥1 次变更
save 300 100       # 300 秒内 ≥100 次变更
save 60 10000      # 60 秒内 ≥10000 次变更
```

任一条件满足 → 自动触发**后台 BGSAVE**。判断依据是 `rdb_changes_since_last_save`（写命令计数，读不计数）。

**实测**：写 50 个 key，计数 39 → 89；执行 `BGSAVE` 后计数归 0。

**自动触发演示**（临时把条件调敏感，测完恢复）：

```bash
CONFIG SET save "1 10"        # 临时：1 秒内 10 次变更即触发
# 快速写 12 个 key → 2 秒后 rdb_changes_since_last_save=0、rdb_last_save_time 更新
CONFIG SET save "3600 1 300 100 60 10000"   # 恢复
```

### 1.2 其他触发

| 触发 | 命令/场景 | 说明 |
|---|---|---|
| 手动后台 | `BGSAVE` | 不阻塞主线程（fork 子进程写） |
| 手动同步 | `SAVE` | **阻塞**主线程直到写完，仅紧急/停机用 |
| 主从全量同步 | 从库首次连接 | 主库 fork 生成 RDB 发给从库 |
| 优雅关闭 | `SHUTDOWN` | 默认尝试落盘；`SHUTDOWN NOSAVE` 跳过 |

---

## 2. AOF 触发时机（简述，详见 PER-002）

1. **每条写命令实时追加**：`appendfsync` 决定刷盘时机（本机 `everysec` 每秒 fsync；`always` 每条；`no` 交 OS）；
2. **AOF 重写**：`auto-aof-rewrite-percentage 100` + `min-size 32MB`——文件增长翻倍且 ≥32MB 自动重写（或手动 `BGREWRITEAOF`）。

**实测**：写 10 个命令，`aof_current_size` 28347135 → 28347487（+352B，实时追加）；`BGREWRITEAOF` 后 28MB → 1KB，`appendonlydir/` 生成新部件 `appendonly.aof.4.base.rdb + incr.aof + manifest`（8.x 多部件原子切换）。

---

## 3. 落盘前的前置步骤（机制）

### 3.1 RDB/BGSAVE 流程

```text
满足 save 条件 / 手动 BGSAVE
  → fork() 子进程（写时复制 COW：子进程拥有内存快照，父子共享页，只有写入的页才复制）
  → 子进程序列化数据写 dump.rdb（不影响主线程服务）
  → 完成后通知父进程：更新 rdb_last_save_time、rdb_changes_since_last_save 归 0
```

- **fork 有耗时**（`latest_fork_usec`）：fork 瞬间主线程要复制页表，大内存实例可能卡几十~几百 ms（OBS-001 实测本机 394us）；
- **COW 内存放大**：快照期间若有大量写，父子进程各自持有被修改页 → 内存短暂上升（"快照内存放大"）；
- 与 PG 对照：RDB ≈ `pg_basebackup` 的物理备份（快照），但 Redis 是 fork+COW 免停机快照，PG 用 WAL 归档 + 备份。

### 3.2 AOF 流程

```text
写命令 → 内存 AOF buffer →（appendfsync 策略）刷盘到 incr.aof
AOF 重写 → fork 子进程生成新 base RDB（当前最终状态）+ 记录期间增量 → manifest 原子切换
```

> 8.x 的 AOF 是"base RDB + incr AOF + manifest"多部件，重写不再是单文件重写，而是**生成新 base + 切换 manifest**——加载时按 manifest 读 base 再回放 incr（见 ENV-002 §1.1.1）。

---

## 4. 阻塞分析速查

| 操作 | 阻塞？ | 说明 |
|---|---|---|
| `BGSAVE` / `BGREWRITEAOF` | 仅 fork 瞬间 | fork 耗时 = `latest_fork_usec` |
| `SAVE` | 全程阻塞 | 生产禁用，用 BGSAVE |
| AOF `everysec` fsync | 每次刷盘小停顿 | 一般无感；磁盘慢时关注 |
| AOF `always` | 每条命令都 fsync | 最安全也最慢 |
| 大 key 写入触发 COW | 写时复制页 | 快照期间写放大 |

> 判断：`INFO persistence` 的 `rdb_bgsave_in_progress` / `aof_rewrite_in_progress` 是否长期=1；`latest_fork_usec` 是否秒级。

---

## 5. 小结

- RDB 触发：自动 save 条件（3 条）、手动 BGSAVE/SAVE、主从全量、SHUTDOWN；
- AOF 触发：每条命令追加（fsync 策略）+ 自动/手动重写；
- 前置机制：RDB=fork+COW 子进程写快照；AOF=缓冲+fsync，重写=新 base+manifest 切换；
- 阻塞点：fork 瞬间与 SAVE 全程，日常盯 `rdb_bgsave_in_progress` / `latest_fork_usec`。

**后续深化**：AOF 三种 fsync 策略、崩溃恢复流程、与 PG WAL 对照进入 **PER-002**；fork/COW 源码级深挖待 **ARC-001**。
