# PG ↔ Redis 基础命令对照

> PostgreSQL 18.4（127.0.0.1:54184，`/data/pgdata/pgdata18.4`）↔ Redis 8.6.2（127.0.0.1:6379）
>
> 目标：给 PG/Oracle DBA 一份"开机就能查"的双端命令对照表。所有命令均为本机实测。

---

## 1. 两个实例速览

| | PostgreSQL 18.4 | Redis 8.6.2 |
| --- | --- | --- |
| 二进制 | `/usr/local/pgsql/pgsql18.4/bin/psql` | `/usr/local/bin/redis-cli` |
| 连接 | `psql -h 127.0.0.1 -p 54184 -d postgres` | `redis-cli -h 127.0.0.1 -p 6379 -a 123456` |
| 就绪检查 | `pg_isready -h 127.0.0.1 -p 54184` | `redis-cli ping` → `PONG` |
| 数据目录 | `/data/pgdata/pgdata18.4`（多个 database） | `/data/redis/data`（16 个逻辑 DB） |

---

## 2. 连接与登录

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 命令行登录 | `psql -h 127.0.0.1 -p 54184 -d postgres -U postgres` | `redis-cli -h 127.0.0.1 -p 6379 -a <password>` |
| 环境变量 | `PGHOST` `PGPORT` `PGDATABASE` `PGUSER` `PGPASSWORD` | `REDISCLI_AUTH`（避免 `-a` 明文） |
| 断开 | `\q` | `quit` / `Ctrl+C` |

要点：

- 本实例端口是 **54184**（PG 18.4 命名规则：3 位补 54，如 17.3→54173），不是 30100；
- Redis `-a` 传密码会在进程列表/历史里明文暴露，生产用 `REDISCLI_AUTH` 或 ACL；
- Redis 没有"用户名+库"连接串概念，认证是实例级（`requirepass`/ACL 用户）。

---

## 3. 实例状态与版本

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 版本 | `SELECT version();` | `INFO server` → `redis_version:8.6.2` |
| 存活检查 | `pg_isready -h 127.0.0.1 -p 54184` | `redis-cli ping` → `PONG` |
| 主备状态 | `SELECT pg_is_in_recovery();` | `INFO replication` → `role:master` |
| 运行时长/进程 | `SELECT pg_postmaster_start_time();` | `INFO server` → `uptime_in_seconds` |

实测：

```text
$ pg_isready -h 127.0.0.1 -p 54184
127.0.0.1:54184 - accepting connections

$ redis-cli ping
PONG
```

---

## 4. 数据库 ↔ 逻辑 DB / 键空间

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 列出数据库 | `\l` / `SELECT datname FROM pg_database;` | `INFO keyspace`（各 DB key 数） |
| 切换 | `\c otherdb` / `psql -d otherdb` | `SELECT 0` ~ `SELECT 15` |
| 数量 | 实例内可建任意多个 database | 固定 16 个（`databases 16`，可配置） |

实测 PG 库列表：`buf_study, postgres, regression, template0, template1, testzex, testzex1`。

关键差异：**PG 的 database 是独立命名空间（表/权限/事务独立）；Redis 的 DB 只是键空间隔离**，
`SELECT` 切库，但复制、持久化、`FLUSHALL` 都是整实例级别——生产多租户不用 DB 号隔离，用不同实例/ACL。

---

## 5. 表 ↔ Key：枚举与结构查看

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 列出表 | `\dt` / `SELECT tablename FROM pg_tables WHERE schemaname='public';` | `SCAN 0 MATCH 'user:*' COUNT 100` |
| 全量枚举 | `\dt`（生产安全） | `KEYS *`（**生产禁用**，阻塞；本实例已 rename 掉） |
| 结构 | `\d table` | `TYPE key` + `OBJECT ENCODING key` + `MEMORY USAGE key` |
| 数量 | `SELECT count(*) FROM pg_tables ...` | `DBSIZE`（当前 DB 的 key 数） |

实测：

```text
PG:  SELECT tablename FROM pg_tables WHERE schemaname='public';   -- (空)
Redis: SCAN 0 MATCH 'dat003:*' COUNT 10
0
dat003:cmd:counter
dat003:cmd:user
```

---

## 6. 数据操作（DML）

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 写入 | `INSERT INTO t VALUES (1,'a');` | `SET k v` / `HSET h f v` |
| 读取 | `SELECT * FROM t WHERE id=1;` | `GET k` / `HGETALL h` |
| 修改 | `UPDATE t SET name='b' WHERE id=1;` | `SET k v2`（覆盖）/ `HSET h f v2` |
| 删除 | `DELETE FROM t WHERE id=1;` | `DEL k` / `HDEL h f` |
| 批量 | `INSERT ... VALUES (...),(...)` | `MSET` / `MGET` / pipeline |
| 计数 | `SELECT count(*) FROM t;` | `SCARD`/`HLEN`/`LLEN`/`ZCARD` |

实测（PG 临时表）：

```text
CREATE TABLE
INSERT 0 2
 id | name
----+------
  1 | a
```

> 心智转换：PG 用 WHERE 条件"筛行"，Redis 用 **Key/类型命令**"直接取"——没有查询优化器，
> 数据怎么组织（String/Hash/ZSet...）决定了能怎么查（见 DAT-001/002）。

---

## 7. 事务

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 开始 | `BEGIN;` | `MULTI` |
| 提交 | `COMMIT;` | `EXEC` |
| 回滚 | `ROLLBACK;`（撤销全部） | `DISCARD`（丢弃排队命令，**不撤销已执行**） |
| 保存点 | `SAVEPOINT sp; ROLLBACK TO sp;` | 无 |
| 隔离级别 | READ COMMITTED / REPEATABLE READ / SERIALIZABLE | 无（单线程逐条执行） |

实测：

```text
PG:   BEGIN; INSERT ... (3,'c'); SELECT count → 3
      ROLLBACK; SELECT count → 2        # 未提交写入被撤销

Redis: MULTI → SET tx v1 → INCR counter → EXEC
       GET tx = v1, GET counter = 1      # 打包顺序执行，全部生效
       MULTI → SET x 1 → DISCARD → EXISTS x = 0
```

关键差异：PG 事务有**回滚 + 隔离 + 一致性**；Redis `MULTI/EXEC` 只是**保证一批命令不被其他客户端插入**
（且 `EXEC` 时某条命令报错不影响其他命令执行），没有回滚语义。

---

## 8. 帮助系统

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 命令语法 | `\h SELECT` | `help @string`（按组）/ `help SET` |
| 全部命令 | `\?` | `COMMAND DOCS` / `COMMAND COUNT` |

实测：

```text
PG:    \h SELECT → Command: SELECT, Description: retrieve rows from a table or view
Redis: help @string → APPEND / DECR / GET / SET ...（带 since 版本号）
```

---

## 9. 权限

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 查看用户 | `\du` / `SELECT rolname FROM pg_roles;` | `ACL LIST` |
| 创建用户 | `CREATE ROLE app LOGIN PASSWORD '...';` | `ACL SETUSER app on >pass ~app:* +get +set` |
| 授权 | `GRANT SELECT ON t TO app;` | ACL 命令 + key 前缀 + 分类授权（`+@read`） |

实测（本实例均为最小配置）：

```text
PG:    rolname → postgres
Redis: user default on ... ~* &* +@all
```

> Redis 6+ 的 ACL = 用户 + 密码 + key 前缀（`~app:*`）+ 命令分类（`+@string`），
> 对标 PG 的 role + 库级/表级权限，粒度是"命令 + key 模式"（详见 SEC-001）。

---

## 10. 配置查看

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 查看参数 | `SHOW port;` → `54184` | `CONFIG GET maxmemory`（**本实例已被禁用**） |
| 配置文件 | `SHOW config_file;` | `redis.conf`（`INFO` 无对应字段，直接看文件） |

实测：

```text
PG:    SHOW port → 54184；SHOW config_file → /data/pgdata/pgdata18.4/postgresql.conf
Redis: CONFIG GET maxmemory → ERR unknown command 'CONFIG'
       （本实例 redis.conf 里 rename-command CONFIG ""，生产常见安全加固）
```

> 同一逻辑：生产 PG 也会限制 `ALTER SYSTEM`/超级用户访问；Redis 用 `rename-command` 禁用
> `CONFIG/KEYS/FLUSHALL` 等危险命令，两者都是"运维命令不下放"的思路。

---

## 11. 备份

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 逻辑备份 | `pg_dump -h 127.0.0.1 -p 54184 -d postgres -f dump.sql` | `redis-cli --rdb dump.rdb` |
| 物理快照 | `pg_basebackup` | `BGSAVE`（fork 生成 dump.rdb） |
| 在线恢复 | replay WAL | 启动加载 RDB/AOF（见 PER 系列） |

实测：

```text
pg_dump --schema-only -f /tmp/pg_dump_demo.sql   → OK（--  PostgreSQL database dump --）
BGSAVE                                           → Background saving started
```

---

## 12. 监控与慢查询

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 活跃会话 | `SELECT * FROM pg_stat_activity;` | `CLIENT LIST` |
| 慢查询 | `log_min_duration_statement` + `auto_explain` | `SLOWLOG GET` / `SLOWLOG LEN` |
| 统计 | `pg_stat_statements` | `INFO stats`（`keyspace_hits/misses`、`expired_keys`） |
| 实时状态 | `\x` + `SELECT pg_stat_*` | `redis-cli --stat`（实时刷新） |

> 对应关系：`pg_stat_activity` ≈ `CLIENT LIST`；PG 慢日志 ≈ `SLOWLOG`；`pg_stat_statements`
> ≈ `INFO commandstats`。详见 OBS 系列。

---

## 13. 心法总结

1. **端口**：PG 实例 54184（版本命名规则），Redis 6379；连接参数结构相同（host/port/db）；
2. **"库"的含义不同**：PG database 是逻辑隔离单元；Redis 16 个 DB 只是键空间编号；
3. **表 ↔ Key 心智**：PG 用 SQL 过滤行，Redis 用 Key + 类型命令直取，无优化器；
4. **事务语义不同**：PG 可回滚，Redis `MULTI/EXEC` 只保证原子排队执行；
5. **生产安全命令**：Redis 的 `KEYS/CONFIG/FLUSHALL` 常被 `rename-command` 禁用——遇到
   "unknown command" 先怀疑安全加固，再怀疑版本差异；
6. **帮助别背**：`\h` 与 `help @<group>` 都是内置手册，随用随查。

## 14. 路径与证据

```text
任务目录  study_record/env/ENV-003_pg_redis_command_mapping/
证据      evidence/connection-status.json     (连接/版本/库列表)
          evidence/dml-transaction.json       (DML + 事务回滚对照)
          evidence/enumeration-help-perm.json (枚举/帮助/权限)
          evidence/config-backup.json         (SHOW vs CONFIG 禁用, pg_dump vs BGSAVE)
          evidence/command-map.json           (完整对照表数据)
PG 实例   /data/pgdata/pgdata18.4（psql -h 127.0.0.1 -p 54184 -d postgres）
Redis 实例 /data/redis（redis-cli -h 127.0.0.1 -p 6379）
```
