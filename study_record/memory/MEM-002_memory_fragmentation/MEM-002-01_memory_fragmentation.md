# MEM-002-01 内存分析与碎片治理

> Redis 8.6.2 / 主库 6379（基线）+ 临时实例 6398（碎片实验）
>
> MEM-001 讲了内存满了怎么淘汰；本文讲**内存"看起来"满了但数据没那么多**的问题：
> 碎片率怎么读、`MEMORY USAGE/DOCTOR` 怎么用、jemalloc 特性、碎片怎么治、容量怎么规划。
> 实测证据：`evidence/fragmentation-source.json`、`fragmentation-experiment.json`。

---

## 0. 一句话心法

> **Redis 的内存账有两本：`used_memory`（逻辑分配，你关心的）和 `used_memory_rss`（进程实际占的，OS 关心的）。**
> 碎片率 = RSS / used。**删除大量 key 后 RSS 不回落 = 碎片**；jemalloc 只保证"内部紧凑"，不保证"及时还给 OS"。

---

## 1. INFO memory 字段怎么读

| 字段 | 含义 | 关注点 |
| --- | --- | --- |
| `used_memory` | 逻辑分配（数据+开销） | 容量规划基准 |
| `used_memory_rss` | 进程实际 RSS | 与 OS 账单/监控对齐 |
| `mem_fragmentation_ratio` | RSS/used 总开销 | **>1.4 且数据量大时警惕** |
| `allocator_frag_ratio` | jemalloc 内部碎片 | >1.1 且碎片字节大→defrag 对象 |
| `allocator_rss_ratio` | 分配器向 OS 多占的页 | 峰值后常见 |
| `rss_overhead_ratio` | 代码段等非堆开销 | 通常稳定 1.0x |
| `used_memory_peak` | 历史峰值 | 解释"为什么删了数据 RSS 还高" |
| `used_memory_dataset_perc` | 数据占（used-startup）比例 | 低说明 overhead 高 |

**第一条 DBA 经验（实测）**：空实例的碎片率毫无意义——

```text
主库 6379：used=1.22MB，RSS=14.04MB → mem_fragmentation_ratio=11.64（虚高）
MEMORY DOCTOR 正确返回："this instance is empty or using very little memory..."
```

RSS 里主要是启动开销（~705KB）+ jemalloc arena 预留页。**看碎片率前先确认有数据**。

---

## 2. 碎片是怎么产生的（实测风暴）

```text
写入 50 万 key（value 10-400B 随机 → jemalloc 多种 size class）
→ 删除 60%（30 万个）

删除前：used=128.58M  rss=144.41M  frag=1.13  allocator_frag=1.00（健康）
删除后：used=54.38M   rss=142.78M  frag=2.63  allocator_frag=2.35（77MB 内部碎片！）
```

- 变长 value 占满 jemalloc 多个 bin；删除后**空闲块散落在各 bin 里**，新写入尺寸不匹配就用不上；
- `MEMORY DOCTOR` 一次报出 3 项：Peak memory（>150%）、High total RSS（>1.4）、High allocator fragmentation（>1.1，建议 activedefrag）。

---

## 3. 治理手段实测对比

### 3.1 MEMORY PURGE：经常没用

```text
MEMORY PURGE（jemalloc 归还空闲页）→ RSS 149MB → 142MB（几乎不变）
```

碎片是**内部布局**问题，不是"有整块空闲页没还"，purge 帮不上忙；DOCTOR 也说 Peak 场景
"harmless"，要么 PURGE 要么重启。

### 3.2 activedefrag：真正有效（但要会配）

```text
CONFIG SET activedefrag yes
60 秒后：allocator_frag 纹丝不动 2.35 ← 为什么？

因为 active-defrag-ignore-bytes 默认 100MB > 本场景碎片 77MB → 不触发！
CONFIG SET active-defrag-ignore-bytes 10MB 后：
t+10s  frag 2.35→1.27   RSS 142MB→106MB
t+20s  frag 1.07        RSS 77MB
t+60s  frag 1.07（稳定） RSS 72MB，DOCTOR 只剩无害的 Peak 提示
```

参数速查（config.c:3135-3289）：

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `activedefrag` | no | 主开关 |
| `active-defrag-threshold-lower/upper` | 10 / 100 | 碎片率低于 10% 不干，100% 全力 |
| `active-defrag-cycle-min/max` | 1 / 25 | CPU 占用 1%-25% 自适应 |
| `active-defrag-ignore-bytes` | **100MB** | 碎片字节低于此不干（**小实例记得调小**） |

### 3.3 终极手段：重启 / 迁移

- 峰值碎片无法接受且业务允许 → **平滑重启**（RDB/AOF 加载后布局重排）；
- 迁移到新实例（临时实例做缓冲）同样能消除历史碎片。

---

## 4. MEMORY USAGE / STATS

```text
MEMORY USAGE key            → 单个 key 逻辑内存（含 robj/编码开销）
MEMORY USAGE key SAMPLES 0  → 嵌套容器全量精确（默认采样 5）
MEMORY STATS                → peak.allocated / total.allocated / fragmentation 等机器可读指标
MEMORY DOCTOR               → 人类可读体检报告（object.c:1421 五条规则）
```

实测：10 万字段 hashtable `SAMPLES 5` 与 `SAMPLES 0` 均为 15,161,847B（此例收敛，嵌套深时可差异）。

---

## 5. 容量规划（结合 MEM-001）

| 项 | 做法 |
| --- | --- |
| 数据预算 | `used_memory_dataset` 起步，按业务峰值 ×1.5~2 |
| 自身开销 | `used_memory_startup`（本机 ~705KB）+ 连接/AOF/复制缓冲（`used_memory_overhead`） |
| 碎片预算 | 常规 1.1-1.3；删除风暴后可到 2+，留 30-50% 余量或开 activedefrag |
| 监控 | `mem_fragmentation_ratio` 与 `used_memory_peak` 一起看，避免误判 |
| 淘汰兜底 | `maxmemory` 按"used 口径"设，碎片会让实际占用超 maxmemory（RSS 不参与淘汰） |

> 注意：**maxmemory 淘汰按 used_memory 计算，碎片不算在里面**——碎片高时实例可能
> "逻辑内存没超限，RSS 却远超"，监控要盯 RSS 与主机内存水位。

---

## 6. 与 PG 对照

| PostgreSQL | Redis |
| --- | --- |
| shared_buffers 固定分配，无碎片概念 | jemalloc 动态分配，碎片是常态 |
| 表膨胀（dead tuples）要 VACUUM | 内存碎片要 activedefrag / 重启 |
| `pg_stat_activity` 看会话内存 | `INFO memory` / `MEMORY STATS` |
| autovacuum 自动治理膨胀 | activedefrag 默认关，需手动开 |
| free/buff/cache 与 PG 无关 | RSS 是容器/主机内存账的关键 |

---

## 7. 小结

1. **碎片率 = RSS/used**，空实例时虚高无意义（实测 11.64），先看有没有数据；
2. **碎片产生**：大写入大删除、变长 value（实测 500k 删 60% → frag 2.63、77MB 内部碎片）；
3. **MEMORY PURGE 治不了内部碎片**（实测几乎无效）；
4. **activedefrag 有效但要配**：默认 ignore-bytes=100MB 是坑，实测调 10MB 后 20 秒 frag 2.35→1.07、RSS 减半；
5. **容量规划**：used + startup + overhead + 碎片余量；maxmemory 不含碎片，监控要盯 RSS。
