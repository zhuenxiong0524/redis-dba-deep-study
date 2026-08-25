# PER-003-01 备份与恢复演练

> Redis 8.6.2 / 临时实例 127.0.0.1:6395-6397（备份、恢复、损坏演练，不碰主库）
>
> PER-001/PER-002 讲了 RDB/AOF 机制；本文把"**出事了怎么把数据找回来**"走一遍：
> 备份 → 校验 → 恢复 → 损坏检测 → AOF 兜底，并对照 PG 的 `pg_basebackup`。
> 实测证据：`evidence/backup-restore-experiment.json`、`backup-restore-source.json`。

---

## 0. 一句话心法

> **Redis 备份 = 拿走 RDB 快照（或 AOF），恢复 = 把文件放回数据目录启动。**
> 没有 PG 那种"WAL 归档 + 任意时间点恢复（PITR）"——Redis 的恢复粒度是**最后一次 BGSAVE + AOF 增量**。
> DBA 职责：**备份要"拷贝到异机"、恢复要"真演练过"、损坏要有"第二份"。**

---

## 1. 备份三件套

### 1.1 BGSAVE 落盘 + 拷贝（主推）

```text
BGSAVE → dump.rdb（1,339,086B，50,003 keys）
cp dump.rdb backups/20260825_2329/dump.rdb   ← 带时间戳，异机/对象存储
```

### 1.2 redis-cli --rdb 远程拉取

```text
redis-cli -p 6396 --rdb remote_backup.rdb
→ Transfer finished with success after 1339169 bytes
→ Checksum OK, 50003 keys read
```

适合不想登录服务器的场景（走客户端协议拉快照）。

### 1.3 备份后立刻校验

```text
redis-check-rdb dump.rdb
→ \o/ RDB looks OK! / 50003 keys read, 1 expires
redis-check-aof appendonly.aof.1.incr.aof
→ AOF is valid, size=3989094, ok_up_to_line=350041
```

> **备份不可信 = 没备份**：每次备份后必须跑 `redis-check-rdb`。

---

## 2. 恢复演练：RDB 恢复到新实例

```text
拷贝 dump.rdb → 新实例数据目录 → redis-server 启动
DBSIZE=50003 | backup:k1 值完整 | bak:hash 字段完整 | bak:ttl TTL=3591s
```

要点：

- **过期元数据一起恢复**（TTL 还在），恢复后到期的 key 会按正常机制删除；
- 恢复用**干净的新目录**，避免旧实例进程还占着文件（"恢复时别开着原实例"）；
- 生产建议恢复到**独立临时实例**先验证，再切换流量。

---

## 3. AOF 兜底：RDB 丢了也能恢复

```text
shutdown → 移走 dump.rdb（模拟 RDB 丢失）→ 重启
日志：DB loaded from base file appendonly.aof.1.base.rdb
      DB loaded from incr file appendonly.aof.1.incr.aof
结果：DBSIZE=50005，连 BGSAVE 之后的 aof:marker / aof:counter=1 都在
```

- AOF 开启时（PER-002），**AOF 优先级高于 RDB**：base.rdb + incr.aof 组合恢复；
- 这就是"RDB 管快、AOF 管少丢"的落地：**RDB 坏了还有 AOF，AOF 坏了还有 RDB，两份都坏才真丢**。

---

## 4. 损坏检测与恢复（实测）

```text
把 dump.rdb 偏移 1000 处改 1 字节
redis-check-rdb → --- RDB ERROR DETECTED --- [offset 1339086] RDB CRC error
用损坏文件启动 → Wrong RDB checksum ... Aborting now → 实例起不来
恢复：用上一份完好备份拷贝回去重启 ✓
```

处置原则：

- 启动报 RDB CRC error = **备份文件坏了**，别原地修，直接换备份；
- `redis-check-aof --fix` 可以截断坏 AOF（丢尾部命令），但**只该当最后手段**；
- 定期"恢复演练"能发现：备份路径没权限、文件被截断、版本不兼容（旧版读不了新版 RDB）等问题。

---

## 5. 与 pg_basebackup 对照

| PostgreSQL | Redis |
| --- | --- |
| `pg_basebackup` 物理备份 | `BGSAVE` / `redis-cli --rdb`（快照） |
| WAL 归档 + PITR（任意时间点） | **无原生 PITR**；AOF 增量兜底到最后一刻 |
| `pg_verifybackup` 校验 | `redis-check-rdb` / `redis-check-aof` |
| 恢复 = 拷贝 PGDATA + 回放 WAL | 恢复 = 放回 dump.rdb（+appendonlydir）启动 |
| 备份/恢复期间不停机 | BGSAVE 后台 fork，不阻塞（PER-001） |
| 逻辑备份 pg_dump | 无 SQL 层逻辑备份（可用 `DEBUG`/scan 导出近似替代） |

---

## 6. DBA 备份清单

| 项 | 要求 |
| --- | --- |
| 备份频率 | RDB：按 RPO 定（如每小时）；开启 AOF everysec 兜底秒级 |
| 保留策略 | 多份 + 时间戳；异机/对象存储至少一份（防主机挂） |
| 校验 | 每次备份后 `redis-check-rdb`；月度恢复演练 |
| 恢复验证 | 恢复到临时实例核对 key 数/抽样/TTL |
| 大实例注意 | BGSAVE fork 内存放大（PER-001）；错峰、监控 `latest_fork_usec` |
| 版本兼容 | 升级 Redis 前先验证旧备份能否被新版本加载 |

---

## 7. 小结

1. **备份**：BGSAVE + 拷贝时间戳目录 / `redis-cli --rdb` 远程拉取，备份后必须校验；
2. **校验**：`redis-check-rdb`（CRC）/ `redis-check-aof`（命令流到第几行）;
3. **恢复**：RDB 放回数据目录启动即恢复，TTL 等元数据完整；
4. **兜底**：AOF 开启时 RDB 丢了也能恢复（base+incr），实测连 BGSAVE 后写入都找回；
5. **损坏**：CRC error = 换备份，不原地修；定期演练是备份体系的灵魂。
