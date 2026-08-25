# MEM-001-01 maxmemory 与淘汰策略

> Redis 8.6.2 / 临时实例 127.0.0.1:6391（不碰主库数据）
>
> 上一篇（ARC-002）讲了数据在内存里的编码；本文讲**内存满了怎么办**：`maxmemory` 上限、10 种淘汰策略、
> 近似 LRU/LFU 的源码机制，以及 DBA 最怕的"淘汰风暴"。全部策略在本机实测。
> 实测证据：`evidence/eviction-source.json`、`policy-experiments.json`、`eviction-storm.json`、`lfu-curve.json`。

---

## 0. 一句话心法

> **Redis 是"内存数据库"：内存满有两种结局——`noeviction` 直接拒绝写入（OOM 报错），
> 或者按策略**悄悄删掉你的 key**（evicted_keys 上涨）。**
> 生产上"被淘汰"比"写不进去"更隐蔽、更危险：**数据没了，但 Redis 看起来还活着。**

---

## 1. 策略全景：8.x 有 10 种（比 7.x 多 2 种）

源码 `config.c:39-48`，按"**淘汰范围 × 淘汰顺序**"两个维度：

| 范围 | 顺序策略 | 含义 |
| --- | --- | --- |
| `volatile-*`（只动带 TTL 的 key） | `volatile-lru` / `volatile-lfu` / `volatile-random` / `volatile-ttl` | 只从 `db->expires` 采样 |
| `allkeys-*`（所有 key） | `allkeys-lru` / `allkeys-lfu` / `allkeys-random` | 从 `db->keys` 采样 |
| 8.x 新增 | **`volatile-lrm` / `allkeys-lrm`** | LRM = Least Recently **Modified**（只记写时间） |
| 兜底 | `noeviction` | 不淘汰，写命令报 OOM |

> 带 `volatile-*` 的坑：**如果所有 key 都没 TTL，内存满了也不淘汰任何 key，直接 OOM**。

---

## 2. 淘汰机制（源码）：采样 → 打分 → 挑最优

`performEvictions()`（evict.c:532）在命令执行路径中同步触发：

```text
内存 > maxmemory ?
 ├─ noeviction → EVICT_FAIL → 当前命令报 OOM
 └─ 循环腾内存：
     evictionPoolPopulate()（evict.c:134）
       ├─ 每个 dict 随机采样 maxmemory_samples 个 key（默认 5）
       ├─ 按策略打分：LRU/LRM=空闲时长 | LFU=255-freq | TTL=ULLONG_MAX-过期时间
       ├─ 插入全局淘汰池（EVPOOL_SIZE=10）
       └─ 取池中最优候选 → 删除 → 继续直到够
     每轮受 evictionTimeLimitUs 时间片限制，避免单命令卡死
```

关键点：

- **近似而非精确**：只采样 `maxmemory-samples`（默认 5）个 key 挑"看起来最该删的"，
  所以叫**近似 LRU/LFU**——代价是吞吐，换来 O(1) 级别的淘汰开销；
- `volatile-*` 采样 `db->expires`（只含 TTL key），`allkeys-*` 采样 `db->keys`（全部）；
- 淘汰是**同步**的：发生在命令处理线程里（ARC-001 的事件循环内），大 key 被淘汰时会卡住循环。

---

## 3. 实测一：noeviction —— 写满即 OOM

```text
maxmemory=1MB, policy=noeviction, 值200B
第 747 个 key：OOM command not allowed when used memory > 'maxmemory'.
evicted_keys: 0 | used_memory: 1,048,552B（钉在 1MB）
```

DBA 要点（实测发现）：

- `used_memory_startup=705,440B`：**maxmemory 包含 Redis 进程自身开销**（事件循环、lua 等），
  1MB 上限里真正给数据的只有 ~240KB——**小内存上限时"可用容量"远比直觉小**；
- 对业务表现：写失败 + 报错，缓存穿透风险立刻出现（OBS-004）。

---

## 4. 实测二：allkeys-lru —— 新 key 优先保留

```text
maxmemory=8MB, 插入 20 万 key（值50B，--pipe 2.7s）
evicted_keys: 134,191 | dbsize: 65,598 | used_memory: 8.00M（钉住）
幸存检查：lru:k1/k50000/k100000 被淘汰，lru:k150000/200000 存活
```

- 最"老"的 key 先被淘汰，最新写入的 key 存活 → LRU 语义正确；
- 20 万 key 只剩 6.5 万 ≈ 每个 key 平均 128B（含 kvobj 开销）——**容量规划要按"带 overhead 的字节"算**。

---

## 5. 实测三：volatile-* 只动带 TTL 的 key

### volatile-lru

```text
10 万带TTL + 5 万不带TTL，maxmemory=8MB
nottl:k1 / nottl:k50000 全部幸存（exists=2）→ TTL key 全被淘汰
```

### volatile-ttl（先过期者优先）

```text
5 万 TTL=60s + 5 万 TTL=3600s + 5 万无TTL
幸存分布：nottl=50,000 | t3600=2,027 | t60=3
```

- 短 TTL 的 key 几乎被淘汰干净 → "**谁的 TTL 先到，先删谁**"；
- 若业务"带 TTL 的 key 是重要数据、不带 TTL 的是垃圾"，volatile 策略会**保护垃圾、牺牲重要数据**——选型要看清范围。

---

## 6. 实测四：LFU —— 热 key 保护（对数计数器）

```text
32MB 插入 10 万 → 热 key 访问 300 次（FREQ=13）→ 调低 8MB
热 k5000 幸存（exists=1, freq=13）| 冷 k90000/k1 被淘汰（freq=5）
```

LFU 计数器实测曲线（`lfu-log-factor=10`，初始 5）：

```text
访问:  1   10   100   300   1000   5000
FREQ:  6    7    10    14     17     41
```

- **对数增长**（evict.c:281 LFULogIncr：越热越难 +1）：高频 key 不会瞬间打满 255，低频 key 有区分度；
- `lfu-decay-time` 控制降温速度（分钟级），热 key 保护适用于"读多写少"的缓存场景；
- 注意：LFU 与 LRU 冲突时，`OBJECT FREQ` 只在 LFU 策略下可用。

---

## 7. 实测五：8.x 新策略 LRM —— 读不保护，写才保护

LRM（Least Recently Modified）只记**最近修改**时间（db.c:61），读取不更新时间戳（db.c:315）：

```text
32MB 插入 10 万 → 读热 k5000 300 次（不修改） + 写热 k100 重新 SET → 调低 8MB
读热 k5000：被淘汰（只读不保护）| 写热 k100：幸存（最近修改）
```

- 适用：**写多读少的缓存**（如计数器/状态类）——LRU/LFU 会把"刚读过但久未写"的 key 留下，
  LRM 则优先淘汰它们；
- 8.x 新增（7.x 无此策略），旧文档里"8 种策略"的说法要更新为 10 种。

---

## 8. 淘汰风暴实测：evicted_keys 飙升

```text
allkeys-lru, maxmemory=4MB, redis-benchmark SET 20 万（-r 随机 key）
本次淘汰：122,907 个 key（8 秒内）
used_memory：3.96M（钉住 4MB，写入持续成功）
SET rps=25,179（vs 无淘汰 3.4 万）→ 淘汰工作占用主线程
风暴中 PING 延迟：max=3ms（淘汰按时间片限流，未完全卡死）
```

DBA 处置清单：

| 信号 | 含义 | 动作 |
| --- | --- | --- |
| `evicted_keys` 持续上涨 | 内存容量不足，正在删数据 | 扩容/清理/降内存占用，查 `used_memory_dataset` |
| `used_memory ≈ maxmemory` 恒成立 | 长期贴着上限 | 容量规划：maxmemory=峰值数据 × 1.5~2 + 自身开销 |
| 淘汰大 key | 同步删大对象卡主线程 | 先拆大 key（OBS-003），再谈策略 |
| 淘汰后延迟抖动 | 时间片限流不够用 | `maxmemory-samples` 调大提高命中率，或降 `maxmemory` |
| 缓存穿透暴增 | 淘汰把缓存打空了 | 策略换 LFU 保护热点；TTL 随机化（OBS-004） |

---

## 9. 与 PG 对照

| PostgreSQL | Redis |
| --- | --- |
| 磁盘满 → 库只读/写失败 | 内存满 → OOM 拒绝 或 静默淘汰 |
| VACUUM/膨胀治理 | 淘汰策略 + 内存碎片治理（MEM-002） |
| work_mem 超限落盘 | maxmemory 超限淘汰（无落盘选项） |
| 无"删数据释放空间"的自动机制 | evicted_keys 是常态机制，不是故障 |
| shared_buffers 是缓存、可重建 | Redis 是数据本体，淘汰=真丢数据 |

---

## 10. 小结

1. **10 种策略** = 范围（volatile/allkeys）× 顺序（LRU/LFU/random/TTL/**LRM**）+ noeviction；
2. **机制**：同步触发 + 采样（maxmemory-samples=5）打分 + 时间片限流，近似而非精确；
3. **实测**：noeviction 写满即 OOM；allkeys-lru 删旧留新；volatile-* 不碰无 TTL key；
   volatile-ttl 先删短 TTL；LFU 保护热 key（对数计数）；8.x LRM 只保护"最近写"的 key；
4. **淘汰风暴**：evicted_keys 飙升 + 内存钉住上限 + 吞吐下降，DBA 要盯 `evicted_keys` 趋势；
5. **maxmemory 含自身开销**（本机 ~705KB）：小内存上限时实际数据容量远小于直觉。
