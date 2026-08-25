# PG ↔ Redis 基础命令与操作对照（教学版）

> PostgreSQL 18.4（127.0.0.1:54184）↔ Redis 8.6.2（127.0.0.1:6379）
>
> 面向 Redis 初学者：每一类 PG 操作，给出**对应的 Redis 写法**，并解释"为什么这么写"。
> 所有命令均为本机实测（证据见 `evidence/`）。

---

## 1. 两个实例速览

| | PostgreSQL | Redis |
| --- | --- | --- |
| 客户端 | `psql -h 127.0.0.1 -p 54184 -d postgres` | `redis-cli -h 127.0.0.1 -p 6379 -a 123456` |
| 就绪检查 | `pg_isready -h 127.0.0.1 -p 54184` | `redis-cli ping` → `PONG` |
| 版本 | `SELECT version();` | `INFO server` → `redis_version:8.6.2` |

> 记法：PG 的"实例/库/表/行/列"，在 Redis 里对应"实例/逻辑DB(一般不切)/key/一个key/字段"。

---

## 2. 新手必读：要不要切库？（SELECT 0..15）

Redis 有 **16 个逻辑 DB**（`databases 16`），命令 `SELECT 5` 切换。实测：

```text
DB0: SET env003:db:demo db0-key    DBSIZE=1    EXISTS env003:db:demo = 1
DB5: DBSIZE=5                      EXISTS env003:db:demo = 0   # 互不可见
INFO keyspace → db0:keys=1  db5:keys=5
```

**结论（直接背）：生产环境不要切库，默认 DB0，用 key 前缀做命名空间。**

原因（与 PG 的关键差异）：

| | PG database | Redis 逻辑 DB |
| --- | --- | --- |
| 隔离能力 | 表/权限/事务完全独立 | 只有 key 空间隔离，**无独立权限/备份/复制** |
| 备份/复制 | 每个库可单独处理 | RDB/AOF、复制、`FLUSHALL` 都是**整实例级** |
| 多租户 | 一个实例多个 database | 用**独立实例**或 Cluster，而不是 DB 号 |
| 对应的概念 | database | **实例/集群** |
| 对应的概念 | schema/表 | **key 前缀**（`user:*`、`order:*`） |

> 例外：本学习环境用 DB5 做实验隔离没问题；真实生产，多业务请分实例。

---

## 3. Redis 的"建表"= Key 设计

PG 要 `CREATE TABLE`，Redis **没有 DDL**——"建表"就是约定 key 的命名和 value 的结构。

```text
PG:   CREATE TABLE t_user(id int primary key, name text, email text, age int);

Redis 的等价"设计"：
  key 前缀  user:          ← 相当于表名 t_user
  key       user:1         ← 相当于一行（主键 id=1 拼进 key）
  列        通过 Hash 的 field 表达  ← 相当于列名
  约束      无（key 天然唯一，重复 SET 即覆盖）
```

**Key 命名规范（冒号分层）**：

```text
user:1                    用户 id=1 的整行（Hash）
user:1:profile            用户 1 的资料（另一个 Hash）
order:20260825:1001       订单（日期分片）
user:1:tags               用户 1 的标签（Set）
rank:202608              8 月排行榜（ZSet）
```

> 心法：**"表"在 Redis 里不是一个对象，而是一类 key 的前缀约定**。这也是为什么
> `SCAN 0 MATCH user:*` 能"列出这张表的所有行"。

---

## 4. 五类型完整 CRUD 对照（重点）

### 4.1 String ↔ PG 的单列标量 / 计数器

| 操作 | PG | Redis |
| --- | --- | --- |
| 插入 | `INSERT INTO kv(k,v) VALUES ('k','v')` | `SET k v` |
| 查询 | `SELECT v FROM kv WHERE k='k'` | `GET k` |
| 更新 | `UPDATE kv SET v='v2' WHERE k='k'` | `SET k v2` |
| 删除 | `DELETE FROM kv WHERE k='k'` | `DEL k` |
| 数值+1 | `UPDATE t SET n=n+1 WHERE id=1` | `INCR k` / `INCRBY k 5` |
| 批量 | `INSERT ... VALUES (...),(...)` | `MSET a 1 b 2` / `MGET a b` |
| 不存在才写 | `INSERT ... ON CONFLICT DO NOTHING` | `SETNX k v` |

```text
PG:   INSERT INTO t_user ... ; UPDATE t SET age=age+1 WHERE id=1;
Redis: INCR user:1:age        # 单线程原子，天然无并发问题
```

### 4.2 Hash ↔ PG 的"一行多列"（最常用）

| 操作 | PG | Redis |
| --- | --- | --- |
| 插入一行 | `INSERT INTO t_user VALUES (1,'Alice','alice@x.com',30)` | `HSET user:1 name Alice email alice@x.com age 30` |
| 查整行 | `SELECT * FROM t_user WHERE id=1` | `HGETALL user:1` |
| 查单列 | `SELECT name FROM t_user WHERE id=1` | `HGET user:1 name` |
| 查多列 | `SELECT name,email FROM t_user WHERE id=1` | `HMGET user:1 name email` |
| 更新 | `UPDATE t_user SET age=31 WHERE id=1` | `HSET user:1 age 31` |
| 删行 | `DELETE FROM t_user WHERE id=2` | `DEL user:2` |
| 删列 | `ALTER TABLE ... DROP COLUMN` | `HDEL user:1 email` |
| 列数 | `SELECT count(*) FROM information_schema.columns ...` | `HLEN user:1`（字段数） |
| 行数 | `SELECT count(*) FROM t_user` | **无内置**，用计数器 `INCR user:count` 或 `SCAN` |

```text
实测：
PG:   SELECT * FROM t_user WHERE id=1 → 1 | Alice | alice@x.com | 30
Redis: HGETALL user:1 → name/Alice/email/alice@x.com/age/30
```

> 注意：`HLEN` 是"一个 key 有多少列"，**不是**"表有多少行"；行数要自己维护计数器。

### 4.3 ZSet ↔ PG 的"排序索引表"（排行榜）

| 操作 | PG | Redis |
| --- | --- | --- |
| 插入 | `INSERT INTO t_rank VALUES ('p1',100)` | `ZADD rank 100 p1` |
| 按分排序 | `SELECT player FROM t_rank ORDER BY score DESC` | `ZREVRANGE rank 0 -1` |
| 前 10 | `... ORDER BY score DESC LIMIT 10` | `ZREVRANGE rank 0 9` |
| 查排名 | `rank() OVER (ORDER BY score DESC)` | `ZREVRANK rank p2` |
| 加分数 | `UPDATE t_rank SET score=score+50 WHERE player='p1'` | `ZINCRBY rank 50 p1` |
| 查分数 | `SELECT score FROM t_rank WHERE player='p1'` | `ZSCORE rank p1` |

```text
实测：
PG:   ORDER BY score DESC → p2, p3, p1；p2 排名=1
Redis: ZREVRANGE rank 0 -1 → p2, p3, p1；ZREVRANK rank p2 → 0（从 0 计）
```

> ZSet = "分数即排序列 + 自带索引"，是 PG 里 `ORDER BY + 索引` 的合体，
> 排行、TopN、延迟队列（score=到期时间戳）都靠它。

### 4.4 Set ↔ PG 的去重 / 集合运算

| 操作 | PG | Redis |
| --- | --- | --- |
| 加标签 | `INSERT INTO t_tag VALUES (1,'a')` | `SADD user:1:tags a` |
| 查全部 | `SELECT DISTINCT tag FROM t_tag WHERE user_id=1` | `SMEMBERS user:1:tags` |
| 判断存在 | `SELECT EXISTS(... WHERE tag='b')` | `SISMEMBER user:1:tags b` |
| 并集 | `SELECT tag FROM t_tag WHERE user_id=1 UNION ...` | `SUNION user:1:tags user:2:tags` |
| 交集 | `... INTERSECT ...` | `SINTER user:1:tags user:2:tags` |
| 数量 | `SELECT count(DISTINCT tag) ...` | `SCARD user:1:tags` |

```text
实测：user1={a,b} user2={b} → SUNION={a,b} SINTER={b} SCARD(user1)=2
```

> Set 的元素天然去重，集合运算在服务端完成，省掉 PG 的 `DISTINCT/UNION/INTERSECT` 往返。

### 4.5 List ↔ PG 的队列 / 时间线

| 操作 | PG | Redis |
| --- | --- | --- |
| 入队 | `INSERT INTO t_queue(msg) VALUES ('m1')` | `RPUSH queue m1` |
| 取前 N | `SELECT msg FROM t_queue ORDER BY id LIMIT 2` | `LRANGE queue 0 1` |
| 出队 | `DELETE ... WHERE id IN (前2条)` | `LPOP queue`（一次一条） |
| 长度 | `SELECT count(*) FROM t_queue` | `LLEN queue` |
| 分页 | `ORDER BY id LIMIT 10 OFFSET 20` | `LRANGE queue 10 29` |
| 阻塞取 | 无（需应用层轮询） | `BLPOP queue 0`（阻塞直到有数据） |

```text
实测：RPUSH m1 m2 m3 → LRANGE 0 1 = m1,m2 → LPOP×2 → LLEN=1 剩 m3
```

> List 是"有序 + 两端操作"结构：`RPUSH/LPOP` 是标准队列，`LRANGE` 天然支持分页。
> 需要"消息不丢 + 多消费组"时升级用 Stream（见 DAT-002）。

---

## 5. SQL → Redis 查询模式速查

| 你熟悉的 SQL | Redis 写法 |
| --- | --- |
| `SELECT * FROM t WHERE id=1` | `HGETALL t:1` / `GET k` |
| `SELECT col FROM t WHERE id=1` | `HGET t:1 col` |
| `WHERE id IN (1,2,3)` | `MGET` / `HMGET` / pipeline |
| `ORDER BY score DESC LIMIT 10` | `ZREVRANGE rank 0 9` |
| 分页 `LIMIT 10 OFFSET 20` | `LRANGE list 10 29` / `ZRANGE rank 10 29` |
| `count(*)` | `SCARD/HLEN/LLEN/ZCARD`；全表行数用计数器或 SCAN |
| `EXISTS(...)` | `EXISTS t:1` |
| `n = n + 1` | `INCR k` / `HINCRBY h f 1` |
| `LIKE 'abc%'` | `SCAN 0 MATCH 'abc*'`（生产慎用，避免全量扫） |
| `ON CONFLICT DO NOTHING` | `SETNX k v` / `HSETNX h f v` |
| 过期清理（TTL） | `EXPIRE k 3600` / `TTL k`（Redis key 级过期是特色） |
| JOIN | **没有**；客户端取多次 / 冗余存储 / Cluster 下同槽 hash tag |
| 聚合 SUM/AVG | 客户端聚合，或 `INCRBY` 维护计数器 / Lua |
| 事务 | `MULTI/EXEC/DISCARD`（无回滚）+ `WATCH` 乐观锁 |

---

## 6. 基础命令速查（连接/状态/权限/配置/备份/监控）

| 操作 | PostgreSQL | Redis |
| --- | --- | --- |
| 列出库/键空间 | `\l` / `SELECT datname FROM pg_database` | `INFO keyspace` |
| 切库 | `\c db` | `SELECT 0..15` |
| 列表/键 | `\dt` | `SCAN 0 MATCH 'user:*'`（`KEYS` 生产禁用） |
| 结构查看 | `\d table` | `TYPE key` + `OBJECT ENCODING key` + `MEMORY USAGE key` |
| 事务 | `BEGIN/COMMIT/ROLLBACK`（可回滚） | `MULTI/EXEC/DISCARD`（打包执行，无回滚） |
| 权限 | `\du` / `CREATE ROLE` / `GRANT` | `ACL LIST` / `ACL SETUSER`（命令+key 前缀粒度） |
| 配置 | `SHOW port;` → 54184 | `CONFIG GET`（本实例已禁用，看 redis.conf） |
| 备份 | `pg_dump` / `pg_basebackup` | `BGSAVE` / `redis-cli --rdb` |
| 慢查询 | `log_min_duration_statement` | `SLOWLOG GET` |
| 活跃连接 | `pg_stat_activity` | `CLIENT LIST` |
| 帮助 | `\h SELECT` / `\?` | `help @string` / `COMMAND DOCS` |
| 退出 | `\q` | `quit` / `Ctrl+C` |

实测要点：

- 本实例 PG 端口 **54184**（规则：3 位补 54，4 位补 5），Redis 6379；
- Redis 的 `CONFIG/KEYS/FLUSHALL` 被 `rename-command` 禁用 → 遇到 `unknown command` 先怀疑安全加固；
- Redis 事务无回滚：`EXEC` 时某条命令报错，其他命令照常执行；`DISCARD` 只丢排队命令；
- `ACL LIST` 实测：`user default on ... ~* &* +@all`（本实例未细分，SEC-001 深入）。

---

## 7. 类型选型速查（一个场景两种写法）

| 业务场景 | PG 习惯 | Redis 选型 |
| --- | --- | --- |
| 用户资料（一行多列） | `t_user` 表 | **Hash**（`user:1`） |
| 简单 KV / 缓存串 | `kv` 表 / 应用缓存 | **String** |
| 计数器/秒杀库存 | `UPDATE ... SET n=n-1` | **String** `DECR` |
| 排行榜 / TopN | `ORDER BY ... LIMIT` | **ZSet** |
| 标签 / 关注关系 | 关联表 + `DISTINCT` | **Set**（`SINTER` 算共同关注） |
| 消息队列 | 队列表 + 轮询 | **List**（`RPUSH/LPOP`）或 **Stream** |
| 会话/登录态 | session 表 | **Hash** + `EXPIRE` |
| UV 去重计数 | `COUNT(DISTINCT uid)` | **HyperLogLog**（误差 0.81%，省 360 倍内存） |

---

## 8. 心法总结（初学者背这 6 条）

1. **不切库**：Redis 16 个 DB 无独立权限/备份/复制，生产默认 DB0，用 key 前缀分层；
2. **建表 = 设计 key**：`user:1`（Hash 一行）、`user:1:tags`（Set）——"表"是前缀约定，不是对象；
3. **查询 = 类型命令**：没有 SQL/优化器，`WHERE id=1` → `HGETALL`，排序 → `ZREVRANGE`，集合 → `SINTER`；
4. **行数要自己维护**：Redis 只答"这个 key 多大"，"表有多少行"用计数器（`INCR user:count`）或 `SCAN`；
5. **事务没有回滚**：`MULTI/EXEC` 只是原子排队；"不存在才写"用 `SETNX`/`WATCH`；
6. **安全命令被禁用是常态**：`KEYS/CONFIG/FLUSHALL` 报 unknown command 先看 `rename-command`。

## 9. 路径与证据

```text
任务目录  study_record/env/ENV-003_pg_redis_command_mapping/
证据      evidence/connection-status.json     (连接/版本/库列表)
          evidence/type-crud.json            (五类型 CRUD 双侧实测)
          evidence/db-switch.json            (切库语义实测)
          evidence/query-pattern.json        (SQL→Redis 模式速查)
          evidence/dml-transaction.json      (DML/事务对照)
          evidence/enumeration-help-perm.json(枚举/帮助/权限)
          evidence/config-backup.json        (配置/备份对照)
          evidence/command-map.json          (完整对照表数据)
PG 实例   psql -h 127.0.0.1 -p 54184 -d postgres（/data/pgdata/pgdata18.4）
Redis 实例 redis-cli -h 127.0.0.1 -p 6379（/data/redis）
```
