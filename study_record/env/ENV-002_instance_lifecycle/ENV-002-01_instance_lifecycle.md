# Redis 实例生命周期与多实例管理

> Redis 8.6.2 / Debian 12 (Linux 5.10.0-38-amd64) / x86_64

ENV-001 完成了单实例（6379）搭建。本任务仿照 PG 的实例管理视角，研究 Redis 实例的
**生命周期**（启动/优雅停止/信号处理/崩溃恢复）、**多实例部署**、**systemd 模板化托管**与**日志轮转**，
并全程与 PostgreSQL 的 `pg_ctl` / systemd 管理方式对照。

---

## 1. 生命周期：启动 → 停止 → 崩溃恢复

### 1.1 启动方式

| 方式 | 命令 | 适用场景 |
| --- | --- | --- |
| 前台 | `redis-server /data/redis/6380/conf/redis.conf` | 排障、调试（日志直达终端） |
| 手动后台 | `... --daemonize yes` | 仅实验场景，生产不推荐 |
| systemd | `systemctl start redis@6380` | 生产推荐（daemonize no 前台 + 托管） |

> 与 PG 对照：`redis-server` 前台 ≈ `postgres -D`；`--daemonize yes` ≈ `pg_ctl start` 的 detach；
> systemd ≈ PG 的 `postgresql@.service` 模板。

### 1.2 优雅关闭：redis-cli shutdown

`redis-cli -p 6380 shutdown` 的日志序列（完整证据见 evidence/lifecycle.json）：

```text
* User requested shutdown...
* Calling fsync() on the AOF file.
* Saving the final RDB snapshot before exiting.
* BGSAVE done, 0 keys saved, 0 keys skipped, 88 bytes written.
* DB saved on disk
* Removing the pid file.
# Redis is now ready to exit, bye bye...
```

要点：

- `shutdown` **默认触发一次最终 RDB 快照 + AOF fsync**，相当于 PG 的 `pg_ctl stop -m fast` 前先做一次检查点；
- `shutdown nosave` 跳过最终快照（场景：确认数据已由 AOF 覆盖，追求快速下线）；
- 关闭过程会正常移除 pidfile。

### 1.3 信号处理

| 信号 | 行为 | 对应 PG |
| --- | --- | --- |
| `SIGTERM` | 与 shutdown 相同：触发优雅退出（日志可见 `Received SIGTERM scheduling shutdown...`） | `pg_ctl stop -m fast` |
| `SIGKILL` | 直接终止，无任何清理（无最终快照、无 pidfile 移除） | `pg_ctl stop -m immediate` |

### 1.4 崩溃恢复实验（kill -9）

```text
写入 6 个 key → kill -9 → 重启
重启日志：
* DB loaded from base file appendonly.aof.1.base.rdb: 0.001 seconds
* DB loaded from incr file appendonly.aof.1.incr.aof: 0.000 seconds
* DB loaded from append only file: 0.002 seconds
* Ready to accept connections tcp
验证：dbsize=6，crash:key:5 => value-5   # 数据完整
```

结论：Redis 8 混合持久化下，SIGKILL 后按 **base RDB + incr AOF** 两段恢复；
`appendfsync everysec` 最坏丢失约 1 秒写入。这与 PG 崩溃后重放 WAL 的心智一致，
但 PG 的 WAL 是"同步/异步提交粒度"保证，Redis 的保证粒度是 fsync 策略。

---

## 2. 多实例管理

### 2.1 目录规范

仿照 PG 每实例独立数据目录的思路，实例以端口为标识：

```text
/data/redis/<port>/
├── conf/redis.conf     # 该实例专属配置
├── data/               # dump.rdb + appendonlydir
├── logs/redis.log
└── redis_<port>.pid
```

6380 实例与 6379 的配置差异**仅**为：`port`、`pidfile`、`logfile`、`dir`、`maxmemory(512mb)`。

### 2.2 端口冲突

在 6380 已占用时再次启动，日志出现：

```text
# Warning: Could not create server TCP listening socket 0.0.0.0:6380: bind: Address already in use
```

> 与 PG 的 `could not bind address ... Address already in use` 完全同源。

**实验发现的陷阱**：冲突失败的进程会在 bind 失败前**先写入 pidfile**，导致 pidfile 指向一个
不存在的进程。因此排查时必须**以 `ps` / `systemctl status` 为准**，pidfile 仅作参考；
发现 pidfile 与进程不一致时用 `systemctl restart` 修正。

---

## 3. systemd 模板化托管

### 3.1 redis@.service 模板

`/etc/systemd/system/redis@.service`，用 `%i` 传实例标识（端口）：

```ini
[Unit]
Description=Redis Server %i
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/redis-server /data/redis/%i/conf/redis.conf
ExecStop=/usr/local/bin/redis-cli -p %i -a 123456 shutdown
Restart=always
RestartSec=2
User=postgres
Group=postgres
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

管理命令：

```text
systemctl enable  redis@6380      # 开机自启
systemctl start   redis@6380
systemctl status  redis@6380
systemctl restart redis@6380
systemctl stop    redis@6380      # 触发 ExecStop -> redis-cli shutdown（优雅）
```

### 3.2 Restart=always 验证

```text
kill -9 2862
journalctl -u redis@6380：
  redis@6380.service: Main process exited, code=killed, status=9/KILL
  Started Redis Server 6380.        # RestartSec=2 后自动拉起
验证：新 pid=2886，pidfile=2886（一致），ping=PONG
```

> 这相当于 PG 用 keepalived / systemd 守护 `postmaster` 的自动拉起，且 Redis 无需像 PG
> 那样处理 stale postmaster.pid——pidfile 由新进程直接重写。

---

## 4. 日志轮转（logrotate）

### 4.1 为什么需要 copytruncate

Redis 在启动时一次性打开 `logfile` 句柄，**不会**像 PG 的 `logging_collector` 那样按时间
自动重开日志文件。因此轮转必须用 `copytruncate`（复制后截断原文件），否则日志句柄仍指向
已改名的文件，磁盘空间无法回收。

### 4.2 配置 /etc/logrotate.d/redis

```text
/data/redis/logs/redis.log /data/redis/6380/logs/redis.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    su postgres postgres
}
```

### 4.3 验证结果

```text
sudo logrotate -f /etc/logrotate.d/redis

/data/redis/logs/    redis.log(0B,新) + redis.log.1(52KB,轮转)
/data/redis/6380/logs/ redis.log(0B,新) + redis.log.1(11.7KB,轮转)
轮转后实例仍可正常写入（ping=PONG）
```

> 与 PG 对照：PG 用 `logging_collector=on` + `log_rotation_age/size` 内置轮转；
> Redis 无内置轮转，标准做法是 logrotate + copytruncate。

---

## 5. 生命周期命令速查（Redis ↔ PostgreSQL）

| 操作 | Redis | PostgreSQL |
| --- | --- | --- |
| 启动 | `systemctl start redis@6380` | `pg_ctl -D ... start` |
| 优雅停止 | `redis-cli shutdown` | `pg_ctl stop -m fast` |
| 强制停止 | `kill -9 <pid>` | `pg_ctl stop -m immediate` |
| 状态检查 | `redis-cli ping` / `systemctl status` | `pg_isready` |
| 实例配置 | `redis.conf`（每实例独立） | `postgresql.conf`（每实例独立） |
| 日志轮转 | logrotate + copytruncate | `logging_collector` |
| 崩溃恢复 | base RDB + incr AOF | WAL replay |

---

## 6. 观察记录与结论

1. 优雅关闭 = AOF fsync + 最终 RDB 快照 + pidfile 移除，序列完整可观测；
2. SIGTERM 与 shutdown 等价；SIGKILL 依赖 AOF 恢复，数据粒度由 `appendfsync` 决定；
3. 多实例成本极低：一份模板 + 独立目录，配置差异只有端口与路径；
4. systemd 模板 + `Restart=always` 是推荐托管方式；pidfile 可能被失败进程污染，以 ps 为准；
5. 日志轮转必须用 `copytruncate`；生产还应把 logrotate 纳入 `logrotate.service` 定时执行检查。

## 7. 路径与证据

```text
实例 6380 配置   /data/redis/6380/conf/redis.conf
systemd 模板     /etc/systemd/system/redis@.service
logrotate 配置   /etc/logrotate.d/redis
证据             evidence/lifecycle.json, multi-instance.json, systemd.json, logrotate.json
```
