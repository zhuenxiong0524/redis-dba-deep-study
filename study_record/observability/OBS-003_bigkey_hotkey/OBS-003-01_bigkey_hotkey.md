# OBS-003-01 大 Key / 热 Key 发现与治理

> Redis 8.6.2 / 127.0.0.1:6379（2026-08-25 环境重建后全功能实例）
>
> 本文解决两个高频故障：**大 Key**（单个 key 过大拖垮单线程）与**热 Key**（单个 key 被打爆）。
> 所有数值均在本机实测（证据：`evidence/bigkey-construct.json`、`bigkey-scan.json`、`bigkey-harm.json`、`hotkey-tracking.json`、`pg-mapping.json`）。

---

## 0. 一句话心法

> **大 Key 是"定时炸弹"：平时不炸，一删除/迁移就卡全局；热 Key 是"单点瓶颈"：一个 key 的热度能打垮整台机器。**
> 发现靠工具扫描，治理靠"拆分 + 分批 + 打散"。

---

## 1. 为什么 Redis 最怕大 Key / 热 Key

- **单线程命令执行**：任何一条命令执行期间，其他命令全部排队。大 key 的操作（删除/迁移/排序）动辄几十上百 ms = 全局卡顿；
- **内存与复制放大**：大 key 占内存不均，主从复制、AOF 重写都要搬运大对象；
- **热 key 单点**：一个 key 访问量巨大时，该 key 所在分片/节点 CPU 与带宽打满，扩容都救不了单个 key。

---

## 2. 构造五类型大 Key 并度量（DB15 隔离库实测）

### 2.1 构造（可复制执行）

```bash
R="redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning -n 15"
$R FLUSHDB

# 大 String：10MB
head -c 10000000 /dev/zero | tr '\0' 'a' > /tmp/bigstr.txt
$R -x SET big:str < /tmp/bigstr.txt

# 大 Hash：10 万字段 / 大 List：50 万 / 大 Set：30 万 / 大 ZSet：20 万（EVAL 循环）
$R EVAL "for i=1,100000 do redis.call('HSET',KEYS[1],'f'..i,'v'..i) end return 1" 1 big:hash
$R EVAL "for i=1,500000 do redis.call('RPUSH',KEYS[1],i) end return 1" 1 big:list
$R EVAL "for i=1,300000 do redis.call('SADD',KEYS[1],i) end return 1" 1 big:set
$R EVAL "for i=1,200000 do redis.call('ZADD',KEYS[1],i,i) end return 1" 1 big:zset
```

### 2.2 度量结果（MEMORY USAGE 实测）

| key | 类型 | 元素 | 编码 | MEMORY USAGE | 观察 |
|---|---|---|---|---|---|
| big:set | set | 30 万 | hashtable | **15.8 MB** | 集合最大：dict 开销高 |
| big:zset | zset | 20 万 | skiplist | **15.0 MB** | skiplist+dict 双结构 |
| big:str | string | 10 MB 值 | raw | 10.5 MB | 大 value 传输贵 |
| big:hash | hash | 10 万字段 | hashtable | 5.7 MB | 字段多也大 |
| big:list | list | 50 万 | quicklist | 2.4 MB | 最省内存，listpack 压缩 |

- `INFO keysizes`（8.x）直接给分布：`strings_sizes:8M=1`、`sets_sizes:8M=1`、`hashes_sizes:4M=1`……一眼看出哪档最大；
- **注意**：8.6.2 的 `MEMORY USAGE` 已不支持 `SAMPLE` 参数（实测语法错误），直接返回完整估算；
- `DEBUG OBJECT big:str` 可看 `serializedlength`（RDB 序列化长度）与 `encoding`，用于粗略评估。

---

## 3. 发现手段：扫描工具实测

### 3.1 `redis-cli --bigkeys`：按元素数/字节数报每类型 top

```bash
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning -n 15 --bigkeys
```

实测摘要（5 个 key 全部命中）：

```text
Biggest   list found "big:list" has 500000 items
Biggest   hash found "big:hash" has 100000 fields
Biggest string found "big:str" has 10000000 bytes
Biggest    set found "big:set" has 300000 members
Biggest   zset found "big:zset" has 200000 members
```

### 3.2 `redis-cli --memkeys`：按 MEMORY USAGE 字节报 top

```bash
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning -n 15 --memkeys
```

实测 top：`set 15780439 > zset 15020440 > str 10485792 > hash 5750742 > list 2477285`（字节）。

**区别（新手易混）**：`--bigkeys` 对 string 报字节、对集合报**元素数**——元素大小差异会误导（1000 个 1KB 的元素比 10000 个 1 字节的占内存多）；`--memkeys` 直接按真实内存字节排，更接近真相。

### 3.3 组合拳建议

```text
--bigkeys       快速找"元素/字节最大"的 key（分钟级，不阻塞：SCAN + O(1) 元数据）
--memkeys       找"真实内存最大"（对每个 key 跑 MEMORY USAGE，O(N)，样本多时慢）
MEMORY USAGE k  单个 key 精确内存
DEBUG OBJECT k  序列化长度与编码（粗略）
INFO keysizes   (8.x) 全局大小分布直方图，最轻量
```

---

## 4. 危害实测：删除大 Key 会卡全局

### 4.1 单条 DEL 阻塞（SLOWLOG 实测）

| 操作 | 耗时 | 说明 |
|---|---|---|
| `DEL big:set`（30 万 hashtable） | **135.6 ms** | 释放 dict 慢，期间全局卡顿 |
| `DEL big:zset`（20 万 skiplist） | 71.8 ms | 双结构释放 |
| `DEL big:hash`（10 万字段） | 53.0 ms | |
| `DEL big:list`（50 万 quicklist） | 4 ms | quicklist 释放快，未进 SLOWLOG |

> 结论：**hashtable 系（set/hash/zset）越大删除越慢，quicklist 最友好**。生产删除大 key 不能直接 DEL。

### 4.2 治理演示：分批删除（SSCAN + SREM 每批 1000）

```bash
# 30 万成员 set，SSCAN 游标循环 + 每批 SREM 1000
cur=0; while :; do
  out=$(redis-cli ... -n 15 --raw SSCAN big:set $cur COUNT 1000)
  cur=$(echo "$out" | head -1); members=$(echo "$out" | tail -n +2 | tr '\n' ' ')
  [ -n "$members" ] && redis-cli ... -n 15 SREM big:set $members >/dev/null
  [ "$cur" = "0" ] && break
done
```

实测：300 批、每批 1~2ms 微停顿（无感），把"一次卡 135ms"拆成"300 次 1ms 抖一下"；本环境 slowlog 阈值 1ms 会记录其中 232 条，**生产默认 10ms 阈值则完全无感**。

### 4.3 大 value 传输也贵
### 4.4 机制：为什么删除会卡？UNLINK 才是正解

- **Redis 没有 MVCC、没有锁**：命令执行是单线程串行的（网络 I/O 8.x 可多线程，执行仍单线程），天然无写冲突，因此也**不需要**并发控制层；
- **代价**：任何一条长命令都是全局停顿——所有会话的命令排队等它执行完。阻塞损失公式：**阻塞时长 × 实例吞吐**（本机约 4 万 QPS，135ms ≈ 5400 个请求被堵）；
- **UNLINK（异步删除）**：从字典摘除 key 是 O(1)，内存由后台线程释放，不阻塞主线程。同一 30 万 set 实测：

```text
DEL    big:set → 155ms，进 SLOWLOG（145ms）——全局卡 145ms
UNLINK big:set → 39ms 返回（含网络往返），SLOWLOG 0 条——内存后台慢慢释放
```

- 观察：UNLINK 后 `used_memory` 立即回落（1.80M），但 RSS/碎片率不立刻归还 OS（`mem_fragmentation_ratio` 短暂飙高 36）——属正常，后台线程释放后由 allocator 回收；
- **结论**：删除大 key 一律用 `UNLINK`；兼容旧版本用分批删除（§4.2）。


`GET big:str`（10MB）单次耗时 **37ms**（回环仍如此）；若 1k QPS 全是 10MB 值，带宽直接打满——大 value 该压缩就压缩。

---

## 5. 热 Key 发现：8.x HOTKEYS 追踪（实测）

### 5.1 新方式：HOTKEYS 命令组（不依赖淘汰策略）

```bash
# 启动追踪：CPU + 网络字节两种指标，top 5，持续 60 秒
redis-cli ... -n 15 HOTKEYS START METRICS 2 CPU NET COUNT 5 DURATION 60

# 制造热点：pipeline 30 万次 GET hot:key，冷 key 只 3 千次
(for i in $(seq 1 300000); do echo "GET hot:key"; done
 for i in $(seq 1 3000);    do echo "GET cold:key"; done) \
  | redis-cli ... -n 15 --pipe >/dev/null

# 取结果（注意：HOTKEYS GET 不带参数）
redis-cli ... -n 15 HOTKEYS GET
redis-cli ... -n 15 HOTKEYS STOP
```

实测结果：

```text
by-cpu-time-us : hot:key 189565us  vs cold:key 1865us   （差 100 倍）
by-net-bytes   : hot:key 7200000  vs cold:key 75000     （差 96 倍）
```

> 8.x 的 HOTKEYS 按 **CPU 时间 / 网络字节** 定位真热点，不依赖淘汰策略，生产可直接用。

### 5.2 老方式：`redis-cli --hotkeys`（必须 *lfu 策略）

```bash
# 前提：maxmemory-policy 必须是 allkeys-lfu / volatile-lfu
CONFIG SET maxmemory-policy allkeys-lfu     # 本实例 CONFIG 已启用
redis-cli ... -n 15 --hotkeys
CONFIG SET maxmemory-policy volatile-lru    # 测完恢复
```

实测：LFU 开启后 5 万次 GET hot:key → `OBJECT FREQ hot:key=115`、`cold:key=0`，`--hotkeys` 输出 `Hot key "hot:key" counter 115`。

### 5.3 两种方式怎么选

```text
HOTKEYS 命令组  8.x 推荐：按 CPU/网络字节，不挑淘汰策略，可定时启动/停止
--hotkeys       仅限 lfu 策略实例；依赖访问频次计数，适合已用 lfu 淘汰的环境
```

---

## 6. 治理方案

### 6.1 大 Key 治理

| 方案 | 适用 | 做法 |
|---|---|---|
| 拆分（分片） | 大 Hash / 大 Set | 按业务维度拆多个 key：`user:1:fields` 拆成 `user:1:part0..9`；写入/读取按 `id % N` 路由 |
| 异步删除 | 必须删的大 key | **首选 `UNLINK`**（Redis 4.0+，O(1) 摘除 + 后台释放，实测 39ms vs DEL 155ms，§4.4） |
| 分批删除/迁移 | 必须删的大 key | SSCAN/SCAN + SREM/HDEL/LREM 每批 500~1000，避免单条 DEL 阻塞（§4.2） |
| 压缩 | 大 String / 大 value | 应用层 gzip/zstd 后写入（实测 4.89MB JSON → gzip 477KB，压缩率 9.5%）；注意 CPU 换带宽的权衡 |
| 换类型 | 大 List 做队列 | 考虑 Stream（8.x 更省）或拆多条队列 |
| 上限约束 | 写入口 | 应用侧限制单 key 大小（如 <1MB），超限告警 |

### 6.2 热 Key 治理

| 方案 | 原理 | 适用 |
|---|---|---|
| 本地缓存（多级缓存） | 客户端/网关缓存热 key，减少直达 Redis | 读多写少的热点 |
| 打散（随机后缀） | `hot:key` → `hot:key:{0..N}` 分摊到多 key/多分片 | 写放大时可接受时 |
| 只读副本分摊 | 从库/副本分担读热点 | 主从架构 |
| 限流/降级 | 热点超阈值时本地降级返回 | 突发热点 |

---

## 7. 与 PG 对照

| Redis | PG | 说明 |
|---|---|---|
| `MEMORY USAGE key` / `DEBUG OBJECT` | `pg_column_size` / `pg_relation_size` | 单值/单表大小度量 |
| `INFO keysizes`（8.x） | `pg_stats` 直方图 / 按 `pg_relation_size` 排序 | 大小分布视角 |
| `--bigkeys` / `--memkeys` | `reltuples` 找大表 / TOAST 大字段排查 | 例行体检 |
| 大 key 删除阻塞（单线程） | 大表 `DROP` / `VACUUM FULL` 长时间锁 | 结构越大维护越贵 |
| 热 key（HOTKEYS） | `pg_stat_user_tables` 热表 + 慢查询频次 | 访问频次视角 |

---

## 8. 小结

- **发现**：`--bigkeys`（元素/字节）→ `--memkeys`（真实内存）→ `MEMORY USAGE`/`DEBUG OBJECT`/`INFO keysizes` 逐层精确定位；
- **危害**：单条 DEL 大 hashtable 实测卡 135ms；分批删除把阻塞拆成 1ms 级微停顿；10MB value 单次传输 37ms；
- **热 key**：8.x `HOTKEYS START/GET` 按 CPU/网络字节定位（推荐）；`--hotkeys` 需 lfu 策略；
- **治理**：大 key 靠拆分/分批/压缩/上限约束；热 key 靠本地缓存/打散/副本分摊/限流。

**后续深化**：大 key 过期、热 key 穿透引发的故障模式进入 **OBS-004 缓存雪崩 / 击穿 / 穿透**。
