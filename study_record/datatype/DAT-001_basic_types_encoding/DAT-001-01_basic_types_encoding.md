# 五大基本类型与内部编码

> Redis 8.6.2 / Debian 12 (Linux 5.10.0-38-amd64) / x86_64 / 实例 127.0.0.1:6379

作为 DBA，理解 Redis 类型不能停留在"命令能跑"的层面：**同一类型在不同数据量/取值下，
底层存储结构完全不同，直接决定内存占用与操作复杂度**。本任务对 String / Hash / List / Set / ZSet
五种基本类型做命令矩阵梳理 + `OBJECT ENCODING` 编码切换实验，并对照 PG 的数据类型心智。

---

## 1. 五种类型命令矩阵

| 类型 | 写入 | 读取 | 查询/统计 | 删除/维护 | 场景 |
| --- | --- | --- | --- | --- | --- |
| String | `SET` `MSET` `SETNX` `INCR` `APPEND` | `GET` `MGET` `STRLEN` | `GETRANGE` | `DEL` `EXPIRE` | 缓存、计数器、分布式锁 |
| Hash | `HSET` `HMSET` `HSETNX` | `HGET` `HMGET` | `HGETALL` `HSCAN` `HLEN` `HEXISTS` | `HDEL` | 对象/实体字段存储 |
| List | `LPUSH` `RPUSH` `LPOP` `RPOP` `LINSERT` | `LRANGE` `LINDEX` | `LLEN` | `LREM` `LTRIM` | 消息队列、时间线 |
| Set | `SADD` `SREM` `SMOVE` | `SMEMBERS` `SRANDMEMBER` `SPOP` | `SISMEMBER` `SCARD` `SINTER` `SUNION` `SDIFF` | `SREM` | 去重、标签、共同好友 |
| ZSet | `ZADD` `ZREM` `ZINCRBY` | `ZRANGE` `ZRANGEBYSCORE` `ZSCORE` | `ZRANK` `ZCARD` `ZCOUNT` | `ZREM` | 排行榜、延迟队列 |

> 与 PG 对照：**Key 就是主键，类型就是"索引方案"**。ZSet 相当于"自带排序索引 + 排名查询"的表，
> Set 相当于"去重索引"，Hash 相当于"宽表的一行"。Redis 没有查询优化器，数据怎么取取决于
> Key 怎么设计——这与"建表规范化"的心智完全相反。

---

## 2. 编码体系总览（Redis 8.x 与 ≤7.x 的关键差异）

`OBJECT ENCODING <key>` 暴露内部存储结构。8.6.2 实测可见编码：

| 类型 | 紧凑编码 | 升级编码 | 8.6.2 默认阈值（redis.conf） |
| --- | --- | --- | --- |
| String | `int` / `embstr` | `raw` | embstr 判定见 §3（8.x 特有） |
| Hash | `listpack` | `hashtable` | 512 项 且 值 ≤64B |
| List | `listpack` | `quicklist` | 节点 ≤8KB（`-2`） |
| Set | `intset`（全整数）/ `listpack`（小字符串集） | `hashtable` | intset 512 项；listpack 128 项 且 ≤64B |
| ZSet | `listpack` | `skiplist` | 128 项 且 成员/分值 ≤64B |

**Redis 8 引入 kvobj（key-value object）**：key 与 value（及过期元数据）尽量内联在**同一块内存**
（源码 `object.h` / `object.c` 的 `kvobjSet`），这改变了 String 编码的判定方式（见 §3），
并让单 key 内存更紧凑。这也是"8.x 老教程说法要甄别"的一个实例：网上大量资料讲
"String ≤44 字节就是 embstr"，在 8.x 上并不准确。

---

## 3. String：int / embstr / raw

### 3.1 实验现象

```text
SET dat001:str:int 12345            -> int        # 规范十进制整数
INCR dat001:str:int                 -> int        # 运算后仍是 int
SET dat001:str:numstr "0012345"     -> embstr     # 前导零，不转 int
SET e "1234...40B"                  -> embstr     # key(1)+val(40)=41
SET r "1234...52B"                  -> raw        # key(1)+val(52)=53
SET dat001:str:v25 "25B"            -> embstr     # key(14)+val(25)=39
SET dat001:str:v44 "44B"            -> raw        # key(14)+val(44)=58
```

### 3.2 关键结论：embstr 边界是「key+value 组合大小」，不是固定 44 字节

8.x 源码 `kvobjSet`（`src/object.c`）的判定：

```c
size = sizeof(kvobj) + keylen + 3 /*key sds头*/ + 4 /*val sds头+结尾*/ + len;
if (size <= CACHE_LINE_SIZE /*64B, x86_64*/)  → 值内联进 kvobj（embstr）
else                                        → 值单独分配 sds（raw）
```

即 **keylen + valuelen ≤ 41 → embstr；≥ 42 → raw**（本实例实测与公式完全一致，见
`evidence/string-encoding.json`）。旧版（≤7.x）是 `OBJ_ENCODING_EMBSTR_SIZE_LIMIT=44`
只看 value 长度；8.x 的 `tryObjectEncoding` 仍按 44 判定，但**落库时 kvobjSet 会按组合大小降级**。

DBA 意义：

- 短 key 不只是省 key 本身的内存，还**影响 value 能否保持 embstr 内联编码**（长 key + 中等 value 会双双变 raw）；
- `int` 编码最省：8 字节指针即值，`MEMORY USAGE` 仅 24B；embstr 单次 64B 分配（40B 值实测 64B）；
  raw 至少两次分配（实测 52B 值 80B）；
- 过期元数据（`EX`）额外 +8B，但不改变 embstr 判定（实测 28B 值 + 14B key 仍按 42 > 41 转 raw）。

---

## 4. Hash / List / Set / ZSet：紧凑编码 → 大结构

### 4.1 Hash：listpack → hashtable

```text
100 字段（小值）              -> listpack
700 字段（小值）              -> hashtable
1 字段但值 100B（>64B）       -> hashtable   # 单个超长值即可触发升级
700 字段 hashtable 删到 10 个 -> 仍 hashtable # 只升不降
```

- 8.x 默认 `hash-max-listpack-entries 512`（**7.x 是 128**），`hash-max-listpack-value 64`；
- 内存：100 字段 listpack = 1031B，101 字段 hashtable（含一个 70B 值）= 4807B，约 **4.7 倍**；
- 空 hash 删空后 key 整体消失（`EXISTS=0`），不存在"空表"。

### 4.2 List：listpack → quicklist

```text
3 个元素            -> listpack
200 元素 × 100B     -> quicklist   # 单节点超 8KB（list-max-listpack-size -2）
```

- 小列表整体是一个 listpack；超过 8KB 后转 quicklist（**双向链表，每个节点仍是 listpack**），
  兼顾两端 `LPUSH/LPOP` 的 O(1) 与内存紧凑；
- 200×100B 实测占用 21029B（约 105B/元素）。

### 4.3 Set：intset / listpack → hashtable

```text
5 个整数成员      -> intset
512 个整数成员    -> intset
513 个整数成员    -> hashtable     # set-max-intset-entries=512
5 个字符串成员    -> listpack
200 个字符串成员  -> hashtable     # set-max-listpack-entries=128
600 整数 hashtable 删到 10 个 -> 仍 hashtable
```

- 整数集合用 `intset`（有序整数数组，二分查找），极省内存；
- 非整数小集合用 listpack（7.2+ 特性）；超限统一升 hashtable；
- `SINTER/SUNION/SDIFF` 这类集合运算在 hashtable 下才高效——**小集合大运算时要注意**。

### 4.4 ZSet：listpack → skiplist

```text
3 个成员    -> listpack
200 个成员  -> skiplist    # zset-max-listpack-entries=128
```

- skiplist 结构 = 跳表（按 score 排序）+ dict（按 member 查 score），是"排序索引"的代价所在：
  100 成员 listpack = 739B，101 成员 skiplist = 9000B，**约 12 倍**；
- 判升级的两个维度：成员数 >128，或任一 member/score >64B。

---

## 5. 编码规则总结（DBA 速记）

1. **只升不降**：写入时超阈值才升级（compact → big）；删除元素**不会**降级，大编码是"粘性"的。
2. **升级是写路径触发的**：读多写少的对象长期保持紧凑编码；写一次超限元素，整对象永久变重。
3. **阈值是"且"关系**：如 Hash 需同时满足 项数 ≤512 且 每值 ≤64B，任一超限即升级。
4. **8.x 特例**：String 的 embstr 判定 = keylen+valuelen ≤ 41（受 kvobj 缓存行内联影响），
   与 ≤7.x 的"value ≤44"不同；`int` 编码始终最优。
5. **空集合自动删除**：Hash/List/Set/ZSet 删空后 key 不存在，注意与 PG"空表仍存在"的差异。

---

## 6. 与 PostgreSQL / Oracle 心智对照

| PostgreSQL / Oracle | Redis | 说明 |
| --- | --- | --- |
| 表 / 行 / 主键 | Key / Value | Key 唯一，天然主键 |
| 类型（text/int/jsonb） | String（int/embstr/raw） | 编码是**内部存储表示**，不是逻辑类型 |
| 索引（B-tree/gin/hash） | 数据类型自带结构 | ZSet=排序索引、Set=去重、Hash=行 |
| TOAST / 行外存储 | raw / 大结构升级 | 都是"大了换个存储结构"的心智 |
| 表膨胀（bloat） | 编码只升不降 | 删除不回收结构，类似膨胀需要重建 |
| `pg_column_size` | `MEMORY USAGE` | 单对象内存诊断 |
| 空表仍存在 | 空集合自动删除 | 逻辑上无"空集合" |
| 规范化设计 | 按访问路径设计 Key | 冗余可接受，索引就是类型本身 |

---

## 7. 观察记录与结论

1. 五类型全部有"紧凑编码 + 升级编码"两级（Set 有三种），`OBJECT ENCODING` 是 DBA 的第一诊断命令；
2. Redis 8.x 的 kvobj 让 String 编码判定变为 key+value 组合大小（实测阈值 41B），与网上 ≤7.x 资料不同；
3. 升级阈值在 8.6.2 默认：hash 512/64、list 8KB、set intset 512 / listpack 128/64、zset 128/64；
4. 编码只升不降 + 空集合自动删除：长生命周期、会反复删改的 key 最终都会落在大编码上，内存估算要按大编码算；
5. skiplist 是最贵的结构（约 12 倍内存），排行榜类数据控制 member 长度与数量对内存影响显著。

## 8. 路径与证据

```text
任务目录  study_record/datatype/DAT-001_basic_types_encoding/
证据      evidence/string-encoding.json  (int/embstr/raw 边界)
          evidence/hash-encoding.json    (listpack→hashtable)
          evidence/list-encoding.json    (listpack→quicklist)
          evidence/set-encoding.json     (intset/listpack→hashtable)
          evidence/zset-encoding.json    (listpack→skiplist)
          evidence/memory-comparison.json(MEMORY USAGE 对比)
          evidence/kvobj-source.json     (kvobjSet 源码取证)
源码      /data/redis_pkg/redis-stable/src/object.c / object.h / config.h
配置      /data/redis/conf/redis.conf（默认阈值未改动）
```
