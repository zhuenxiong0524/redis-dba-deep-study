# Redis 命令练习手册（可复制执行）

> Redis 8.6.2 / 127.0.0.1:6379 / 密码 123456
>
> 本手册是 ENV-003 的配套练习：每条命令都给了**可粘贴执行**的写法与**预期输出**，
> 并标注对应的 PG 操作，方便你边敲边建立"PG 怎么写 → Redis 怎么写"的映射。
> 所有输出均已在本机实测（证据 `evidence/practice-handbook.json`、`pitfalls.json`）。

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

- 本实例（2026-08-25 环境重建后）为方便实验已启用 `KEYS / CONFIG / FLUSHDB / FLUSHALL`
  （旧环境曾 `rename-command` 禁用）；但**生产仍建议**：看 key 用 `SCAN` 代替 `KEYS`、清库优先 `DEL`（见 §10）；
- 练习完统一清理命令见 §10，保持 DB5 干净。

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

## 2.5 重点理解：一行 = 一个 key，"表" = 前缀集合

刚练完 Hash 你可能会问："`HSET user:1 ...` 这不就一行数据吗？"

**对，一次 `HSET user:1` 就是一行。想多几行，就多建几个 key：**

```text
HSET user:1 name Alice age 30   # 第 1 行
HSET user:2 name Bob   age 25   # 第 2 行
HSET user:3 name Carol age 28   # 第 3 行

SCAN 0 MATCH 'user:*'           # → user:1 user:2 user:3（这就是"整张表"）
HGETALL user:2                  # → name Bob age 25（查第 2 行）
```

心智模型：

| PG | Redis |
| --- | --- |
| `t_user` 表 | **所有 `user:` 前缀的 key 合起来**才是"表" |
| `INSERT` 一行 | 一次 `HSET user:N ...` |
| 主键 `id` | 拼进 key 名（`user:1` 的 `1`） |
| "表"是独立对象 | **没有表对象**，只有共享前缀的 key |

两个推论：

1. **每一行是独立对象**：`user:1` 和 `user:2` 互不相干，各有自己的类型/编码/过期/内存；
   字段可以不一样（`user:2` 没有 `email` 不会报错——没有建表约束）；
2. **"一个 key 装多行"也存在**：Hash 是一行一个 key；但 ZSet/Set/List 相反——**一个 key 装很多成员**，
   如排行榜 `ZADD rank 100 p1 200 p2 150 p3` 是 3 个成员共用一个 key。
   选型口诀：**行多列少 → Hash（一行一个 key）；一行多条目 → List/Set/ZSet（一个 key 装一堆）**。

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

### 5.0 先看懂 ZADD 参数：score member 成对出现

```text
ZADD rank 100 p1 200 p2 150 p3
      │    │  │  │  │  │  │  │
      │    └──┴──┴──┴──┴──┴──┘
      │        每两个一组：(score member)
      │        (100 p1) (200 p2) (150 p3)
      └── key 还是键名 = "排行榜"这张表
```

- key 依然是 `rank`；变的只是"值"的形态——ZSet 的值是**多个 `(score, member)` 二元组**；
- `member` = 成员（相当于"行"的唯一标识，PG 的 `player` 列）；
- `score` = 分数（相当于"排序列"的值，PG `ORDER BY` 的那一列）；
- **顺序注意**：PG 写 `INSERT VALUES ('p1', 100)`（先 member 后 score），
  Redis 写 `ZADD rank 100 p1`（**先 score 后 member**），方向相反容易记反；

对照表：

| PG | Redis |
| --- | --- |
| `CREATE TABLE t_rank(player, score)` | key `rank` = 表 |
| `INSERT VALUES ('p1',100)` | `ZADD rank 100 p1` |
| `player` 列 | `member` |
| `score` 列 | `score` |
| `ORDER BY score DESC` | `ZREVRANGE rank 0 -1` |
| `UPDATE ... SET score=300 WHERE player='p1'` | `ZADD rank 300 p1`（同 member 即更新，返回 0） |

> 速记：ZSet = **带分数的 Set**（`SADD tags m` 加成员；ZSet 每个成员前多放一个分数）。


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

## 8. 看懂 Redis 命令：命名规律与参数含义

### 8.1 命名规律：首字母 = 类型

Redis 命令几乎都是"**类型首字母 + 操作**"：

| 首字母 | 类型 | 例子 | 记忆 |
| --- | --- | --- | --- |
| `H` | Hash | `HSET` `HGET` `HGETALL` `HLEN` `HDEL` | Hash Set / Get / ... |
| `L` | List | `LPUSH` `LPOP` `LRANGE` `LLEN` | List Push / Range / Length |
| `S` | Set | `SADD` `SMEMBERS` `SINTER` `SCARD` | Set Add / Members / Intersection / CARDinality |
| `Z` | ZSet | `ZADD` `ZRANGE` `ZSCORE` `ZCARD` | Z 开头专属（skiplist） |
| `X` | Stream | `XADD` `XREAD` `XACK` | 流类型 |
| （无） | String/通用 | `SET` `GET` `INCR` `DEL` `EXPIRE` | 最基础的一组 |

其余字母=操作缩写：`GET`/`SET`/`ADD`/`REM`/`POP`/`PUSH`/`RANGE`/`LEN`/`CARD`/`EXISTS`/`INCR BY`...

```text
HGETALL  = Hash GET ALL           ← 取 Hash 全部字段
LRANGE   = List RANGE             ← 取 List 一段范围
SINTER   = Set INTERsection       ← 集合交集
SUNION   = Set UNION              ← 集合并集
SDIFF    = Set DIFFerence         ← 集合差集
SISMEMBER= Set IS MEMBER          ← 是不是成员
SCARD    = Set CARDinality        ← 基数=去重后数量
ZREVRANGE= ZSet REVerse RANGE     ← 反向范围（降序）
INCRBY   = INCRement BY           ← 加 N
SETNX    = SET if Not eXists      ← 不存在才写
BLPOP    = Blocking LPOP          ← 阻塞式弹出
TTL      = Time To Live           ← 剩余存活时间
```

> 记住这个规律后，看到一个没见过的命令也能猜出七八分。

### 8.2 参数约定：范围类命令（以 `LRANGE` 为例）

```text
LRANGE key start stop
```

- **索引从 0 开始**；
- `start` 和 `stop` **都包含**（闭区间）：`LRANGE list 1 3` → 第 2、3、4 个元素；
- **负数 = 从尾部数**：`-1` 最后一个，`-2` 倒数第二个；
- 常用组合：

| 写法 | 含义 | 等价场景 |
| --- | --- | --- |
| `LRANGE list 0 -1` | 全部元素 | `SELECT *` |
| `LRANGE list 0 9` | 前 10 个 | `LIMIT 10` |
| `LRANGE list 10 29` | 第 11~30 个 | `LIMIT 20 OFFSET 10`（分页） |
| `LRANGE list -2 -1` | 最后 2 个 | 尾部元素 |

实测：`RPUSH list a b c d e` 后，`LRANGE list 1 3` → `b c d`；`LRANGE list -2 -1` → `d e`。

`ZRANGE` 的 `start stop` 也是同样的位置索引；**按分数取**用 `ZRANGEBYSCORE`，区间规则不同：

```text
ZRANGEBYSCORE rank min max
100 150      # 默认闭区间，含 100 和 150
(100 200     # ( 表示开区间，不含 100
-inf +inf    # 全区间（不限定分数）
```

实测：`ZADD rank 100 p1 150 p3 200 p2`，`ZRANGEBYSCORE rank 100 150` → `p1 p3`；
`(100 200` → `p3 p2`（100 被排除）。

### 8.3 手册内全部命令拆解表

| 命令 | 拆解 | 含义 / 参数 |
| --- | --- | --- |
| `SET k v` | SET | 写 String：k=键，v=值 |
| `GET k` | GET | 读 String；不存在返回 `(nil)` |
| `APPEND k v` | APPEND | 尾部追加，返回新长度 |
| `STRLEN k` | STRing LENgth | 字符串字节数 |
| `INCR k` / `INCRBY k n` | INCRement [BY] | +1 / +n（不存在当 0） |
| `DECR k` | DECRemen | -1 |
| `MSET/MGET` | Multi SET/GET | 批量写/读 |
| `SETNX k v` | SET if Not eXists | 不存在才写；返回 0/1 |
| `EXPIRE k s` / `TTL k` | EXPIRE / Time To Live | 设过期秒 / 查剩余秒 |
| `PERSIST k` | PERSIST | 移除过期 |
| `HSET k f v` | Hash SET | 写字段；返回**新增**字段数 |
| `HGET k f` / `HMGET k f1 f2` | Hash GET [Multi] | 取单/多字段 |
| `HGETALL k` | Hash GET ALL | 取全部字段 |
| `HLEN k` | Hash LENgth | 字段数（不是行数！） |
| `HEXISTS k f` | Hash EXISTS | 字段是否存在；0/1 |
| `HINCRBY k f n` | Hash INCR BY | 字段值 +n |
| `HDEL k f` | Hash DEL | 删字段 |
| `RPUSH/LPUSH k v...` | Right/Left PUSH | 尾部/头部入队；返回新长度 |
| `LPOP/RPOP k` | Left/Right POP | 头部/尾部取出（取出即删） |
| `BLPOP k t` | Blocking LPOP | 阻塞弹出；t=超时秒，0=永远等 |
| `LRANGE k s e` | List RANGE | 按位置范围取（闭区间、负索引） |
| `LLEN k` | List LENgth | 列表长度 |
| `LINDEX k i` | List INDEX | 按下标取单个 |
| `LTRIM k s e` | List TRIM | 只保留范围，其余删 |
| `SADD k m...` | Set ADD | 加成员；返回**新增**数（重复=0） |
| `SMEMBERS k` | Set MEMBERS | 全部成员（无序） |
| `SISMEMBER k m` | Set IS MEMBER | 是否成员；0/1 |
| `SCARD k` | Set CARDinality | 成员数（基数） |
| `SREM k m` | Set REMove | 删成员 |
| `SINTER/SUNION/SDIFF` | Set INTER/UNION/DIFF | 交集/并集/差集 |
| `SPOP k` | Set POP | 随机弹出一个成员 |
| `ZADD k s m` | ZSet ADD | 加成员；s=score(分数)，m=member |
| `ZSCORE k m` | ZSet SCORE | 查分数 |
| `ZCARD k` | ZSet CARDinality | 成员数 |
| `ZRANGE/ZREVRANGE k s e` | ZSet [REVerse] RANGE | 按位置升/降序取 |
| `ZRANK/ZREVRANK k m` | ZSet [REVerse] RANK | 排名（从 0 开始） |
| `ZRANGEBYSCORE k min max` | ZSet RANGE BY SCORE | 按分数区间取 |
| `ZINCRBY k n m` | ZSet INCR BY | 分数 +n |
| `ZREM k m` | ZSet REMove | 删成员 |
| `TYPE k` / `OBJECT ENCODING k` | TYPE / OBJECT | 类型 / 内部编码 |
| `EXISTS k` | EXISTS | key 是否存在；0/1 |
| `DEL k...` | DELete | 删除；返回删除个数 |
| `SCAN c MATCH p COUNT n` | SCAN | 增量遍历；c=游标(0结束)，MATCH=前缀模式 |
| `RENAME/COPY k k2` | RENAME/COPY | 改名/复制 |

---

## 9. 新手易混淆 / 易错点（全部实测）


### 8.1 类型不匹配：用错命令直接报错

```text
HSET user:1 name Alice    # OK
GET user:1                # WRONGTYPE Operation against a key holding the wrong kind of value
```

> 每个 key 有固定类型，`GET` 只能取 String。报错先 `TYPE key` 确认类型再选命令。

### 8.2 (nil) 是"没数据"，(error) 才是"命令错了"

```text
GET notexist              # (nil)   ← 正常结果，等价 PG 查询无结果
SET name abc              # OK
INCR name                 # ERR value is not an integer or out of range  ← 命令错误
```

### 8.3 返回值含义：数字 ≠ OK

| 命令 | 返回 | 含义 |
| --- | --- | --- |
| `SET k v` | `OK` | 状态回复 |
| `HSET` 新增字段 | `1` | 新增的字段数 |
| `HSET` 更新已有字段 | `0` | 字段已存在（只是覆盖） |
| `SADD` 新成员 / 重复成员 | `2` / `0` | 实际新增的成员数 |
| `DEL k` | `1` | 删除的 key 个数（删不存在的返回 0） |

> 新手最懵的：`HSET` 更新为什么返回 0？——它返回的是"**新增**了多少"，不是"成功"。

### 8.4 key 区分大小写，命令不区分

```text
SET User:1 carol          # 与 user:1 是两个完全不同的 key！
GET user:1                # (nil)
GET User:1                # "carol"
```

### 8.5 过期是"整 key 级"，不是字段/成员级

```text
HSET sess:1 uid 1         # OK
EXPIRE sess:1 300         # 1（整个 sess:1 300 秒后消失）
TTL sess:1                # 300
```

> 不能对 Hash 的单个 field、ZSet 的单个成员单独设 TTL（7.4+ 有 subexpiry 实验特性，一般别依赖）。
> 想"字段级过期"就拆成独立 key。

### 8.6 排名从 0 开始（PG 的 rank() 从 1 开始）

```text
ZADD rank 100 p1 200 p2 150 p3
ZREVRANK rank p2          # 0 ← 第一名是 0！
```

> 展示给业务方时记得 +1，这是排行榜最常见的"差 1" bug 来源。

### 8.7 列表负索引：-1 是最后一个

```text
RPUSH list a b c
LRANGE list 0 -1          # a b c（取全部，-1=末尾）
LRANGE list -2 -1         # b c（取最后两个）
```

### 8.8 再说一遍"行"：Hash 一行一个 key，ZSet/Set/List 一个 key 多行

```text
HSET user:1 ... / HSET user:2 ...   # 3 行 = 3 个 key
ZADD rank 100 p1 200 p2 ...         # 3 个"行"共用一个 key rank
```

---

## 10. 练习后清理

```text
# 删除单个 key
DEL name counter site1 site2 site3 user:1 tags:a tags:b rank queue q2

# 或者：SCAN 枚举当前库所有 key 后批量 DEL（先看后删，安全）
SCAN 0 MATCH '*' COUNT 100
DEL <列出来的 key...>
```

> 注意：本实例已启用 `FLUSHDB/FLUSHALL`（旧环境曾禁用）；教学上仍建议用 `DEL` 精确清理，避免误清。
> 生产上禁止对别人的库随意 FLUSH；练习库删自己建的 key 即可。

---

## 11. 对照速记（本手册核心）

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

## 12. 路径与证据

```text
文章      study_record/env/ENV-003_pg_redis_command_mapping/ENV-003-02_redis_command_practice.md
证据      evidence/practice-handbook.json（全部命令实测输出）
          evidence/pitfalls.json（易错场景实测）
实例      redis-cli -h 127.0.0.1 -p 6379 -a 123456 -n 5
```
