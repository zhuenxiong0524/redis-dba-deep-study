# 高级类型：Bitmap / HyperLogLog / Geo / Stream

> Redis 8.6.2 / Debian 12 (Linux 5.10.0-38-amd64) / x86_64 / 实例 127.0.0.1:6379

DAT-001 覆盖了五大基本类型。本任务研究四种**高级类型**：它们不是新的"表结构"，而是**针对特定
访问模式的专用数据结构**——Bitmap 是位数组、HyperLogLog 是概率计数器、Geo 是地理索引、
Stream 是内存消息日志。DBA 关注点：什么场景必须用它们、内存能省多少、有什么局限。

---

## 1. 四种高级类型总览

| 类型 | 本质 | 核心命令 | 典型场景 | 内存特点 |
| --- | --- | --- | --- | --- |
| Bitmap | String 上的位操作 | `SETBIT` `GETBIT` `BITCOUNT` `BITPOS` `BITOP` `BITFIELD` | 签到、在线状态、布隆过滤 | 1 位/状态，10 万用户 ≈12.5KB |
| HyperLogLog | 概率基数统计（16384 个 6bit 寄存器） | `PFADD` `PFCOUNT` `PFMERGE` | UV、独立访客去重 | 固定 ≈12KB，与基数无关 |
| Geo | ZSet + geohash 编码 | `GEOADD` `GEOPOS` `GEODIST` `GEOHASH` `GEOSEARCH` | 附近的人、LBS | 同 ZSet |
| Stream | 内存日志（rax 树 + listpack） | `XADD` `XREAD` `XGROUP` `XREADGROUP` `XACK` `XPENDING` `XAUTOCLAIM` | 消息队列、事件流 | 按消息数线性增长 |

---

## 2. Bitmap：位数组（String 的位视图）

### 2.1 签到场景实验

```text
SETBIT sign:202608 1 1 / 3 1 / 5 1   # 用户ID(位偏移) 第1/3/5天签到
BITCOUNT sign:202608                  # = 3（8月签到总天数）
BITPOS  sign:202608 1                 # = 1（第一个签到日）
STRLEN  sign:202608                   # = 1B（偏移到 7，仅用 1 字节）
```

- **BITCOUNT**：统计 1 的个数（签到天数/在线人数）；**BITPOS**：找第一个 1/0（首次签到）；
- **BITOP AND/OR/XOR/NOT**：位运算做「连续签到」（两天 AND）、「活跃交集」：
  实测 day1={1,3,5,7,9} ∩ day2={3,5,7,11} → {3,5,7}，`BITCOUNT=3`；
- **BITFIELD**：一次读写多个位域（如把签到月拆成 4 位/域），Redis 6.2+ 还支持有符号整数位域。

### 2.2 内存优势（实测）

| 方案 | 数据量 | 内存 |
| --- | --- | --- |
| Bitmap | 10 万用户 1 位/人 | 14,376B（约 0.14B/用户，含 kvobj 开销） |
| String key | 1000 个状态 key | ≈32KB（单 key 32B × 1000） |

**10 万用户的状态，比 1000 个普通 String key 还省**。本质：Bitmap 把"10 万个独立 key 的元数据开销"
压缩成"1 个 key 的位数组"。

> 与 PG 对照：相当于把 `user_status` 表按 user_id 的位图索引；`BITOP` 类似位图索引的 AND/OR 加速。

---

## 3. HyperLogLog：基数统计（UV）

### 3.1 实验：10 万独立用户

```text
PFADD uv:2026-08-25 user:1 ... user:100000   # 再重复加 5 万已有用户
PFCOUNT uv:2026-08-25                        # = 99471（真实 100000）
PFMERGE uv:week1 uv:day1 uv:day2 ...         # 合并多天
```

### 3.2 关键指标

- **误差**：实测 0.529%，理论标准误差 0.81%——**固定误差，与基数大小无关**；
- **内存**：HLL 10 万用户 = 14,368B（≈12KB 固定，16384 寄存器 × 6bit）；
  同样的 Set = 5,161,847B（5.2MB）——**约 1/360**；
- **PFMERGE**：合并两天（10万 + 10万，重叠 5 万）→ 149,696（真实 150,000，误差 0.2%）。

### 3.3 局限（DBA 必须知道）

- HLL **只能统计基数**：能回答"有多少独立用户"，**不能**回答"哪些用户"；
- **不支持删除单元素**：PFADD 不能移除某个 user，统计口径变了只能重建 key（或用新 key + 定期合并）；
- 小基数时（<几百）误差感知明显，Redis 内部用稀疏表示缓解。

> 与 PG 对照：相当于 `COUNT(DISTINCT user_id)` 的**近似版**（PG 的 `- -` 抽样统计/扩展如
> `hyperloglog` 扩展思路一致），适合 UV 大屏这类"不需要精确值"的场景。

---

## 4. Geo：地理坐标索引

### 4.1 实验：附近的门店

```text
GEOADD shops 116.397 39.909 "天安门" 116.407 39.916 "故宫" ...
GEODIST shops 天安门 故宫 km        # 1.1548km
GEOSEARCH shops FROMMEMBER 天安门 BYRADIUS 10 km ASC WITHCOORD WITHDIST
# → 故宫 1.15km / 颐和园 5.39km / 北京南站 6.71km（上海外滩被排除）
```

### 4.2 底层真相

- `TYPE shops` → **zset**；`ZSCORE shops 天安门` → `4069885365177207`（**geohash 整数**）；
  所以 Geo 没有新存储结构，是 ZSet 的 score 换成了经纬度的 geohash 编码；
- **经纬度转 geohash 有精度损失**：`GEOPOS` 返回的是还原值（实测 116.397 → 116.3970002532），
  精确到米级足够，但**不适合需要亚米精度的场景**；
- Redis 6.2+ 推荐 `GEOSEARCH`（FROMMEMBER / BYLONLAT + BYRADIUS / BYBOX），旧 `GEORADIUS*` 已过时。

---

## 5. Stream：消息队列 / 事件日志

### 5.1 基础：追加式日志

```text
XADD orders * order-id 1001 user u1     # 返回 ID：1787633650279-0
XADD orders MAXLEN 3 * event e1 ...     # 有界日志：只保留最新 3 条
XLEN orders                             # 长度
XRANGE orders - + COUNT 2               # 按范围读
XREAD BLOCK 200 STREAMS orders $        # 阻塞读（超时返回 nil）
```

- 消息 ID = `毫秒时间戳-序列号`，天然有序、全局唯一，**不需要像 PG 序列那样单独建 sequence**；
- `MAXLEN` 裁剪让 Stream 可当**有界事件日志**用（等价"只留最近 N 条"）。

### 5.2 消费组：消息不丢失的关键

```text
XGROUP CREATE orders g1 0      # 组 g1 从头消费
XGROUP CREATE orders g2 $      # 组 g2 只消费新消息（多组互不干扰）
XREADGROUP GROUP g1 c1 COUNT 2 STREAMS orders >   # c1 消费 2 条
XPENDING orders g1             # 已投递未确认：2 条（owner=c1）
XACK orders g1 <id> <id>       # 确认后 pending=0
XAUTOCLAIM orders g1 c2 0 0-0  # c2 接管 c1 超时未确认的消息（消费者宕机恢复）
```

### 5.3 Stream vs List 队列（实测差异）

| 维度 | List + BRPOP | Stream + XREADGROUP |
| --- | --- | --- |
| 消费后 | **弹出即删**（LLEN 3→2） | 消息仍在 stream（XLEN 不变） |
| 多消费者 | 一条消息只能被一个消费者取走 | **多消费组各自独立进度**（广播/订阅） |
| 消息确认 | 无（取走即认为成功） | `XACK` 显式确认，未确认进 PEL 可重放 |
| 宕机恢复 | 已弹出的消息丢失 | `XPENDING` + `XAUTOCLAIM` 接管 |
| 适用 | 简单任务队列 | 需要可靠性/多组消费的消息系统 |

> 与 PG 对照：Stream ≈ 一个"追加式日志表 + 多个游标（消费组）"；`XPENDING` ≈ 未提交事务列表，
> `XACK` ≈ commit，`XAUTOCLAIM` ≈ 检测到死事务后接管。

---

## 6. 高级类型选型速查（DBA 版）

1. **状态/签到类**（每一位一个用户/一天）→ Bitmap，空间最省，位运算免费；
2. **只要去重计数、不要明细**（UV）→ HyperLogLog，12KB 搞定百万级基数；要明细就用 Set；
3. **附近的人/门店** → Geo，本质是 ZSet，无新结构；注意 geohash 精度损失；
4. **需要可靠消费/多组订阅的消息** → Stream（XACK/PEL/消费组）；简单队列用 List 即可；
5. **共同陷阱**：HLL 不能删元素、Stream 消息不自动清理（要用 MAXLEN 或 XTRIM）、
   Geo 的坐标是近似还原值。

## 7. 路径与证据

```text
任务目录  study_record/datatype/DAT-002_advanced_types/
证据      evidence/bitmap-encoding.json   (签到+BITOP+内存对比)
          evidence/hll-encoding.json      (10万UV误差率+Set对比+PFMERGE)
          evidence/geo-encoding.json      (GEOSEARCH+ZSet底层)
          evidence/stream-encoding.json   (消费组+XACK+XAUTOCLAIM+MAXLEN)
源码      /data/redis_pkg/redis-stable/src/t_stream.c / geohash.c / hyperloglog.c
配置      /data/redis/conf/redis.conf
```
