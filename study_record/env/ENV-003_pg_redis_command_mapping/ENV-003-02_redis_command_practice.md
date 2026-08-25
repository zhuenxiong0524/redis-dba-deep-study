# Redis 命令练习手册（可复制执行）

> Redis 8.6.2 / 127.0.0.1:6379 / 密码 123456
>
> 本手册是 ENV-003 的配套练习：每条命令都给了**可粘贴执行**的写法与**预期输出**，
> 并标注对应的 PG 操作，方便你边敲边建立"PG 怎么写 → Redis 怎么写"的映射。
> 所有输出均已在本机实测（证据 `evidence/practice-handbook.json`）。

---

## 0. 环境准备与清理

```bash
# 进入交互模式（建议开一个终端专门练）
redis-cli -h 127.0.0.1 -p 6379 -a 123456 -n 5
```

- `-n 5`：用逻辑库 DB5 练习，不碰业务数据（为什么不用切库见 ENV-003-01 §2）；
- 先验证连接：

```text
127.0.0.1:6379[5]> PING
PONG
127.0.0.1:6379[5]> DBSIZE        # 当前库 key 数，应为 0
0
```

- 本实例的 `KEYS / CONFIG / FLUSHDB / FLUSHALL` 已被 `rename-command` 禁用，
  所以"看有哪些 key"用 `SCAN`，"清库"用 `DEL`（见 §8）；
- 练习完统一清理命令见 §8，保持 DB5 干净。

---

## 1. String：对标 PG 的单值 / 计数器

```text
# ---------- 插入 / 查询 / 覆盖 ----------
SET name Alice            # OK
GET name                  # "Alice"
SET name Alice2           # OK（覆盖，PG: UPDATE ... SET v='Alice2'）
GET name                  # "Alice2"

# ---------- 追加 / 长度 ----------
APPEND name "!"           # 7（返回新长度）
STRLEN name               # 7

# ---------- 计数器（PG: UPDATE t SET n=n+1） ----------
SET counter 10            # OK
INCR counter              # 11
INCRBY counter 5          # 16
DECR counter              # 15

# ---------- 批量（PG: INSERT 多行 / SELECT WHERE id IN） ----------
MSET site1 redis site2 pg # OK
MGET site1 site2          # 1) "redis"  2) "pg"

# ---------- 不存在才写（PG: INSERT ... ON CONFLICT DO NOTHING） ----------
SETNX site2 oracle        # 0（已存在，不写）
SETNX site3 mysql         # 1（不存在，写入）

# ---------- 过期（PG 无对应，Redis 特色） ----------
EXPIRE name 300           # 1（设置 300 秒后过期）
TTL name                  # 300
PERSIST name              # 1（去掉过期）
TTL name                  # -1（-1=永不过期；-2=key 不存在）
```

> 练习完看编码：`OBJECT ENCODING counter` → `int`；`OBJECT ENCODING name` → `raw`
> （`APPEND` 后重建为 raw，这是 Redis 8 的行为细节，见 DAT-001）。

---

## 2. Hash：对标 PG 的"一行多列"（最常用）

```text
# ---------- 插入一行（PG: INSERT INTO t_user VALUES (1,'Alice','alice@x.com',30)） ----------
HSET user:1 name Alice email alice@x.com age 30   # 3（新增 3 个字段）
HSET user:1 city Beijing                          # 1

# ---------- 查询 ----------
HGET user:1 name            # "Alice"（PG: SELECT name FROM t_user WHERE id=1）
HMGET user:1 name email     # 1) "Alice"  2) "alice@x.com"
HGETALL user:1              # name/Alice/email/alice@x.com/age/30/city/Beijing
HLEN user:1                 # 4（字段数，不是行数！）
HEXISTS user:1 email        # 1（PG: EXISTS(SELECT 1 ...)）

# ---------- 更新 ----------
HSET user:1 age 31          # 0（字段已存在，更新返回 0）
HGET user:1 age             # "31"
HINCRBY user:1 age 5        # 36（PG: UPDATE ... SET age=age+5）

# ---------- 删除 ----------
HDEL user:1 city            # 1
HGETALL user:1              # 只剩 name/email/age
DEL user:1                  # 1（删整行；PG: DELETE FROM t_user WHERE id=1）
```

> 练习完看 `OBJECT ENCODING user:1` → `listpack`（小 hash 的紧凑编码，DAT-001）。

---

## 3. List：对标 PG 的队列 / 时间线 / 分页

```text
# ---------- 入队（PG: INSERT INTO t_queue(msg) VALUES (...)） ----------
RPUSH queue msg1 msg2 msg3  # 3（长度）
LRANGE queue 0 -1           # 1) "msg1" 2) "msg2" 3) "msg3"
LRANGE queue 0 1            # 前 2 条（PG: ORDER BY id LIMIT 2）
LLEN queue                  # 3（PG: SELECT count(*)）

# ---------- 两端操作 ----------
LPUSH queue msg0            # 4（头插）
LRANGE queue 0 -1           # msg0,msg1,msg2,msg3
LPOP queue                  # "msg0"（从头部取出并删除）
RPOP queue                  # "msg3"（从尾部取出并删除）

# ---------- 按位置/裁剪 ----------
LINDEX queue 0              # "msg1"（PG 无直接对应）
LTRIM queue 0 0             # OK（只保留第 0~0 条）
LRANGE queue 0 -1           # "msg1"

# ---------- 阻塞取（PG 需应用层轮询） ----------
BLPOP q2 3                  # 有数据立即返回：1) "q2" 2) "job1"
BLPOP noqueue 2             # 空队列阻塞 2 秒后返回 (nil)
```

> 组合套路：`RPUSH` 入队 + `BLPOP` 出队 = 生产者/消费者队列；`LRANGE` 就是分页查询。

---

## 4. Set：对标 PG 的去重 / 集合运算

```text
# ---------- 插入（自动去重） ----------
SADD tags:a redis mysql     # 2
SADD tags:b mysql pg        # 2

# ---------- 查询 ----------
SMEMBERS tags:a             # 1) "mysql" 2) "redis"（无序）
SISMEMBER tags:a redis      # 1（PG: EXISTS(SELECT ... WHERE tag='redis')）
SCARD tags:a                # 2（PG: SELECT count(DISTINCT tag)）

# ---------- 增删 ----------
SADD tags:a pg              # 1（新成员）
SREM tags:a pg              # 1（移除）

# ---------- 集合运算（PG: UNION / INTERSECT / EXCEPT） ----------
SINTER tags:a tags:b        # "mysql"（共同）
SUNION tags:a tags:b        # "mysql" "redis"（并集）
SDIFF tags:a tags:b         # "redis"（差集）

# ---------- 随机取 ----------
SPOP tags:a                 # 随机弹出一个（如 "redis"）
SADD tags:big 1 2 3 4 5     # 5（支持批量/数字）
SCARD tags:big              # 5
```

> 练习：两个人共同关注的用户 → `SINTER user:1:follow user:2:follow`。

---

## 5. ZSet：对标 PG 的排行榜 / TopN

```text
# ---------- 插入（member + score） ----------
ZADD rank 100 p1 200 p2 150 p3   # 3
ZCARD rank                       # 3（成员数）

# ---------- 查询 ----------
ZSCORE rank p1                   # "100"（PG: SELECT score WHERE player='p1'）
ZRANGE rank 0 -1                 # 按分数升序：p1,p3,p2
ZREVRANGE rank 0 -1              # 按分数降序：p2,p3,p1（排行榜）
ZREVRANGE rank 0 1               # Top2：p2,p3

# ---------- 排名（PG: rank() OVER (ORDER BY score DESC)） ----------
ZRANK rank p3                    # 1（升序排名，从 0 计）
ZREVRANK rank p3                 # 1（降序排名：p2 是 0，p3 是 1）

# ---------- 按分数区间 ----------
ZRANGEBYSCORE rank 100 150       # p1,p3（100<=score<=150）

# ---------- 更新 / 删除 ----------
ZINCRBY rank 50 p1               # "150"（PG: UPDATE ... SET score=score+50）
ZSCORE rank p1                   # "150"
ZREM rank p2                     # 1
ZCARD rank                       # 2
```

> 心法：ZSet 的 score 可以是任意可排序值——排行榜用分数，**延迟队列用到期时间戳**
> （score=now+delay，`ZRANGEBYSCORE ... 0 now` 取到期任务）。

---

## 6. Key 通用命令（任何类型都能用）

```text
EXISTS rank          # 1（key 是否存在）
TYPE rank            # "zset"（PG: 查看对象类型）
TYPE user:1          # "hash"
OBJECT ENCODING rank # "listpack"（内部编码，DAT-001 深入）
RENAME rank rank2    # OK（改名）
COPY rank2 rank3     # 1（复制，Redis 6.2+）
GET name3            # "Alice2!"（验证 COPY 内容）
DEL rank2 rank3      # 2（批量删除）

# 枚举 key（PG: \dt 列表）——生产禁用 KEYS，用 SCAN：
SCAN 0 MATCH 'user:*' COUNT 100   # 返回游标 0 + 匹配的 key
SCAN 0 MATCH '*' COUNT 100        # 看当前库所有 key
```

> `SCAN` 第一行返回值是"下一次游标"，0 表示遍历结束；它是增量遍历，不阻塞（`KEYS` 才阻塞）。

---

## 7. 综合练习：一个小业务（用户 + 标签 + 排行榜 + 队列）

把前面学的串起来，模拟一个"用户系统"：

```text
# 用户表（Hash）——PG: t_user
HSET user:1 name Alice age 30
HSET user:2 name Bob   age 25
HSET user:3 name Carol age 28

# 用户标签（Set）——PG: t_user_tag
SADD user:1:tags redis mysql
SADD user:2:tags mysql
SADD user:3:tags redis pg

# 用户活跃榜（ZSet）——PG: 按 score 排序查询
ZADD rank:202608 30 user:1 25 user:2 28 user:3

# 站内信队列（List）——PG: t_notify
RPUSH notify:1 "welcome" "new feature"

# ---------- 现在来"查询" ----------
HGETALL user:1                      # 用户1整行
SMEMBERS user:1:tags                # 用户1的标签
SINTER user:1:tags user:3:tags      # 用户1和用户3的共同标签 → redis
ZREVRANGE rank:202608 0 1           # 8月活跃 Top2 → user:1 user:3
ZREVRANK rank:202608 user:3         # 用户3排名（0起）→ 1
LPOP notify:1                       # 取用户1的一条站内信 → "welcome"
```

练习完成后自测：把"查用户2的标签""取 8 月排行榜第 3 名""用户2和用户3共同标签"
翻译成上面的命令写出来。

---

## 8. 练习后清理

```text
# 删除单个 key
DEL name counter site1 site2 site3 user:1 tags:a tags:b rank queue q2

# 或者：SCAN 枚举当前库所有 key 后批量 DEL（先看后删，安全）
SCAN 0 MATCH '*' COUNT 100
DEL <列出来的 key...>
```

> 注意：本实例 `FLUSHDB/FLUSHALL` 被禁用（`rename-command`），所以"清库"只能用 DEL。
> 生产上禁止对别人的库随意 FLUSH；练习库删自己建的 key 即可。

---

## 9. 对照速记（本手册核心）

| 想做的事 | PG | Redis 命令 |
| --- | --- | --- |
| 存一个值 | `INSERT INTO kv ...` | `SET k v` |
| 计数器 | `SET n=n+1` | `INCR k` |
| 一行多列 | `INSERT INTO t_user ...` | `HSET user:1 f v ...` |
| 查整行 | `SELECT * WHERE id=1` | `HGETALL user:1` |
| 排行榜 TopN | `ORDER BY score DESC LIMIT 10` | `ZREVRANGE rank 0 9` |
| 去重/交集 | `DISTINCT / INTERSECT` | `SADD` / `SINTER` |
| 队列 | 队列表+轮询 | `RPUSH` / `BLPOP` |
| 过期 | 无 | `EXPIRE k 300` |
| 枚举 key | `\dt` | `SCAN 0 MATCH 'prefix:*'` |
| 清理 | `DELETE` | `DEL k` |

## 10. 路径与证据

```text
文章      study_record/env/ENV-003_pg_redis_command_mapping/ENV-003-02_redis_command_practice.md
证据      evidence/practice-handbook.json（全部命令实测输出）
实例      redis-cli -h 127.0.0.1 -p 6379 -a 123456 -n 5
```
