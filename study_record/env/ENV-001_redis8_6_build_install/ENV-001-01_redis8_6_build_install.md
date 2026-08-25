# Redis 8.6.2 源码获取、编译与安装

> Redis 8.6.2 / Debian 12 (Linux 5.10.0-38-amd64) / x86_64

作为 PostgreSQL DBA 迁移学习 Redis 的第一步，本文仿照 PG 的 ENV-001 思路，记录 Redis 源码环境、
实例目录、配置文件与启动方式，并与 PG 环境结构做对照。

---

## 1. 为什么需要源码编译

与 PostgreSQL 一样，使用源码编译而非包管理器安装，核心原因一致：

- **调试符号（Debug Symbols）**：源码编译默认带 `-g -ggdb`，二进制 `with debug_info, not stripped`，
  后续可以用 GDB 跟踪命令执行路径（如 `getCommand` → `lookupKeyRead`），复刻 PG 的 GDB 研究方法；
- **内存分配器可控**：Redis 对内存管理敏感，生产建议 `jemalloc`，源码编译可指定 `MALLOC=jemalloc`；
- **版本精确可控**：锁定 8.6.2，后续 RDB/AOF 文件格式、命令行为都以该版本为准。

## 2. 环境信息

| 项目 | 值 |
| --- | --- |
| 操作系统 | Debian 12 (Linux 5.10.0-38-amd64) |
| 架构 | x86_64 |
| Redis 版本 | 8.6.2（`REDIS_VERSION`，见 `src/version.h`） |
| 编译工具 | gcc 10.2.1 |
| 编译选项 | `DEBUG=-g -ggdb`、`OPTIMIZATION=-O3`、`MALLOC=jemalloc` |
| 运行用户 | postgres（非 root，与 PG 实例规范一致） |
| 实例端口 | 6379 |
| 源码目录 | /data/redis_pkg/redis-stable |
| 二进制路径 | /usr/local/bin/redis-server |

## 3. 源码获取

源码位于 `/data/redis_pkg/redis-stable`，版本定义：

```c
#define REDIS_VERSION "8.6.2"
```

二进制版本验证：

```text
Redis server v=8.6.2 sha=00000000:1 malloc=jemalloc-5.3.0 bits=64 build=126f3c4486452d73
```

## 4. 实例目录结构

仿照 PG 的 `/data/pgdata/pgdata18.4` 思路，Redis 实例独立目录：

```text
/data/redis/
├── conf/redis.conf      # 配置文件（相当于 postgresql.conf）
├── data/                # 数据目录：dump.rdb + appendonlydir（相当于 PGDATA 数据区）
│   ├── dump.rdb
│   └── appendonlydir/
│       ├── appendonly.aof.1.base.rdb
│       └── appendonly.aof.1.incr.aof
├── logs/redis.log       # 服务日志（相当于 pg 日志）
└── redis_6379.pid       # PID 文件（postgres 可写路径）
```

## 5. 配置文件关键项

`/data/redis/conf/redis.conf` 有效配置摘要（完整见 evidence/redis-config.json）：

| 配置项 | 值 | DBA 视角说明 |
| --- | --- | --- |
| `bind 0.0.0.0` / `port 6379` | 监听地址与端口 | 生产建议绑定内网地址 |
| `requirepass 123456` | 访问密码 | 学习环境口令；生产建议用 ACL |
| `appendonly yes` | 开启 AOF | 相当于 WAL，崩溃后按 AOF 恢复 |
| `appendfsync everysec` | fsync 频率 | 默认折中：最多丢 1 秒数据 |
| `maxmemory 1280mb` | 内存上限 | 数据全在内存，容量即内存 |
| `maxmemory-policy volatile-lru` | 淘汰策略 | 仅淘汰带 TTL 的 key |
| `enable-debug-command local` | 允许 DEBUG 命令（仅回环） | 2026-08-25 环境重建后启用，便于实验；生产建议 no |
| `latency-monitor-threshold 100` | 延迟事件监控阈值(ms) | 2026-08-25 重建后启用；旧版曾 rename 禁用 CONFIG/KEYS/FLUSH* |
| `slowlog-log-slower-than 10000` | 慢日志阈值(µs) | 相当于 pg 慢查询 |

## 6. systemd 托管与启动

仿照 PG 实例管理思路，Redis 由 systemd 托管（`/etc/systemd/system/redis.service`）：

```ini
[Service]
ExecStart=/usr/local/bin/redis-server /data/redis/conf/redis.conf
ExecStop=/usr/local/bin/redis-cli -a 123456 shutdown
Restart=always
User=postgres
Group=postgres
LimitNOFILE=65535
```

要点：

- `User=postgres`：与 PG 一致，数据库服务不以 root 运行；
- `Restart=always`：进程退出自动拉起（相当于 PG 的 keepalive/服务守护）；
- `daemonize no`：前台运行交给 systemd 管理，等价于 `postgres -D` 前台方式。

常用管理命令（类比 PG）：

| Redis | PostgreSQL |
| --- | --- |
| `systemctl start/stop/restart redis` | `systemctl start/stop/restart postgresql` |
| `redis-cli shutdown` | `pg_ctl stop` |
| `redis-cli ping` | `pg_isready` |
| `redis-cli info server` | `SELECT version();` |

## 7. 功能验证

```text
$ redis-cli -a 123456 ping
PONG
$ redis-cli -a 123456 set study:env:hello "hello redis 8.6"
OK
$ redis-cli -a 123456 get study:env:hello
"hello redis 8.6"
```

- 进程：`postgres` 用户运行，pidfile `/data/redis/redis_6379.pid` 正常写入；
- 服务状态：`systemctl is-active redis` → `active`；
- 启动日志：`DB loaded from base file appendonly.aof.1.base.rdb` → `Ready to accept connections tcp`；
- 慢日志：`SLOWLOG len` = 0。

## 8. 环境对照：Redis vs PostgreSQL

| 维度 | PostgreSQL 18.4 | Redis 8.6.2 |
| --- | --- | --- |
| 源码 | /data/myfuture/src*（PG 源码树） | /data/redis_pkg/redis-stable |
| 数据目录 | /data/pgdata/pgdata18.4 | /data/redis/data |
| 配置文件 | postgresql.conf | redis.conf |
| 服务用户 | postgres | postgres |
| 端口 | 5432 | 6379 |
| 崩溃恢复 | WAL replay | AOF / RDB 加载 |
| 内存管理 | shared_buffers | maxmemory + 淘汰策略 |

## 9. 观察记录（供后续任务）

- 空库小数据量时 `mem_fragmentation_ratio` 高达 19.75（RSS 15.8MB / used 0.8MB），属正常现象：
  进程刚启动、数据量极小时 RSS 被代码段与 jemalloc 保留区占据，评估碎片应在大数据量下进行；
- `rdb_changes_since_last_save` 不为 0 说明有变更未触发快照，后续 PER-001 深入 RDB 触发条件。

## 10. 路径汇总

```text
二进制          /usr/local/bin/redis-server
配置文件        /data/redis/conf/redis.conf
数据目录        /data/redis/data
日志            /data/redis/logs/redis.log
PID 文件        /data/redis/redis_6379.pid
systemd 单元    /etc/systemd/system/redis.service
源码            /data/redis_pkg/redis-stable
证据            evidence/env-info.json, redis-config.json, verification.json
```
