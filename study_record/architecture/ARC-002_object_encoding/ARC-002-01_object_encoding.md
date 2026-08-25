# ARC-002-01 对象系统与内存编码实现

> Redis 8.6.2 / 127.0.0.1:6379
>
> ARC-001 讲了"命令谁在执行"；本文讲"**数据在内存里长什么样**"——`robj` 对象系统 + 六种内存编码，
> 并用实测回答 DBA 最关心的内存问题：**为什么 Hash 多塞 2 个字段内存翻 4 倍？为什么 Set 多 1 个整数内存翻 20 倍？**
> 实测证据：`evidence/object-system-source.json`、`encoding-experiment.json`、`collection-encoding.json`、`config-thresholds.json`。

---

## 0. 一句话心法

> **Redis 的"类型"是逻辑概念（String/Hash/...），"编码"是物理实现（int/embstr/raw/intset/listpack/...）。**
> 同一个 key 可以随时从省内存的紧凑编码切换到功能全的通用编码——**切换发生的那一瞬间，内存暴涨**。
> DBA 记住一件事：**小数据用紧凑编码（省 4~20 倍内存），超阈值就"打回原形"。**

---

## 1. robj：一切值的统一外壳

源码 `object.h:99`，每个 value 都是 `struct redisObject`（别名 robj），16 字节头 + 数据指针：

```c
struct redisObject {
    unsigned type:4;        /* OBJ_STRING / OBJ_LIST / OBJ_SET / OBJ_ZSET / OBJ_HASH / ... */
    unsigned encoding:4;    /* RAW INT HT INTSET SKIPLIST EMBSTR QUICKLIST STREAM LISTPACK */
    unsigned refcount:23;   /* 引用计数（2^23-1 = 共享对象标记，永不销毁） */
    unsigned iskvobj:1;     /* 8.x：是否 kvobj（键值一体） */
    unsigned metabits:8;    /* 8.x：过期等元数据位图 */
    unsigned lru:24;        /* LRU 时钟 / LFU 频次（淘汰策略用） */
    void *ptr;              /* 指向实际数据：sds / intset / dict / quicklist / zskiplist ... */
};
```

要点：

- **type 与 encoding 分离**：`TYPE key` 看 type，`OBJECT ENCODING key` 看 encoding；同一 type 可换 encoding；
- **引用计数**：对象被多个 key 引用时共享（`incrRefCount`/`decrRefCount`），`OBJECT REFCOUNT` 可查；
- **8.x 的 kvobj**：key 直接内嵌进对象（`object.h:99` 注释区），value 小也内嵌——这就是 DAT-003 里
  "过期元数据 +8B"、以及本文后面 embstr 阈值变化的原因；
- 16 字节头对千万级 key 是真实开销：**短 key + 短 value 时，头部占比极大**（实测见 §3）。

---

## 2. 编码图谱：紧凑 → 通用

| 类型 | 紧凑编码（省内存） | 通用编码（功能全） | 切换方向 |
| --- | --- | --- | --- |
| String | `int` / `embstr` | `raw` | 变长/变复杂时升级 |
| Hash | `listpack`（字段+值紧凑排列） | `hashtable`（dict） | 超 512 字段/64B 值 |
| Set | `intset`（有序整数数组）/ `listpack`（字符串） | `hashtable` | 超 512 整数 或 128 字符串/64B |
| ZSet | `listpack` | `skiplist`（跳表）+dict 双结构 | 超 128 成员/64B |
| List | `quicklist`（listpack 节点链表） | ——（无整体升级） | 节点分裂/压缩 |

各底层结构源码：

- `intset`（intset.h:35）：`encoding + length + contents[]`，有序整数数组，二分查找；
- `listpack`（listpack.c:27）：6 字节头 + 紧凑字节数组，整数最多 9 字节编码，**一个 malloc 一块地**；
- `quicklist`（quicklist.h:107）：双向链表，每个节点是一个 listpack（默认 ≤8KB），可 LZF 压缩中部节点；
- `skiplist`（server.h:1699）：`zset = dict + skiplist`——dict 按 member 查 score，跳表按 score 排序取范围。

---

## 3. String 的三种编码（8.x 有变化）

### 3.1 INT / EMBSTR / RAW 实测

```text
SET s:int 100        → enc=int     mem=24B   ← 值就是指针本身，零数据分配
SET s:bigint 100000  → enc=int     mem=32B   ← 超出共享范围，仍是 int 编码
SET s:emb hello      → enc=embstr  mem=40B   ← robj+SDS 一次 malloc
SET s:raw44 <44字符> → enc=raw     mem=80B   ← 两次 malloc（robj + sds）
SET s:raw45 <45字符> → enc=raw     mem=88B
```

### 3.2 8.x 的 embstr 阈值：不再是固定的 44

7.x 规则是值 ≤44 字节就是 embstr；8.x 因 kvobj 把 key 也内嵌，受 **64B 缓存行**约束
（源码 `kvobjSet`，object.c：`size <= CACHE_LINE_SIZE` 才内嵌）。实测三个键长：

```text
key_len=1  → value≤40 为 embstr，41 起 raw     （1+40=41）
key_len=5  → value≤36 为 embstr，37 起 raw     （5+36=41）
key_len=10 → value≤31 为 embstr，32 起 raw     （10+31=41）
```

> **8.x 规则：key_len + value_len ≤ 41 时值才保持 embstr。** 所以 8.x 上"44 字节内必是 embstr"
> 的旧知识已不成立——键越长，值越早变 raw。

### 3.3 共享整数：8.x 不再用于 keyspace

7.x 里 `SET k 5` 后 `OBJECT REFCOUNT k` 会显示 `2147483647`（共享整数，0..9999 预建）。
8.x 实测三个 key 都存 5：

```text
OBJECT REFCOUNT k1/k2/k3 → 1（每 key 独立 kvobj，不再共享）
```

`OBJ_SHARED_INTEGERS=10000`（server.h:125）仍存在，但只用于内部回复等场景；
keyspace 值由 kvobj 承载后各持一份。**这是 8.x 对"共享对象"模型的重大调整。**

---

## 4. 集合编码切换：阈值触发的内存跳变（实测）

### 4.1 Hash：511→512 没事，513 翻 4.6 倍

```text
511 字段: listpack   5,947 B
512 字段: listpack   5,959 B   （阈值 hash-max-listpack-entries=512）
513 字段: hashtable 27,550 B   ← 4.6x 跳变
值 64B:  listpack / 值 65B: hashtable   （hash-max-listpack-value=64）
```

### 4.2 Set：整数多 1 个翻 21.8 倍

```text
512 个整数:     intset     1,056 B
513 个整数:     hashtable 23,041 B   ← 21.8x！intset 超限直接转 HT（t_set.c:63），不经过 listpack
128 个字符串:   listpack     691 B
129 个字符串:   hashtable   5,890 B   ← 8.5x（set-max-listpack-entries=128）
成员 65B:       hashtable（set-max-listpack-value=64）
```

### 4.3 ZSet：129 个成员翻 12.5 倍

```text
127 成员: listpack    939 B
128 成员: listpack    948 B   （阈值 zset-max-listpack-entries=128）
129 成员: skiplist 11,752 B   ← 12.5x（dict+跳表双结构，t_zset.c:55）
```

### 4.4 List：不会"整体升级"

```text
100,000 元素: quicklist 465,762 B（约 4.66B/元素）
```

List 永远是 quicklist（listpack 节点链表），元素多了只是**节点分裂**，不存在整表变 hashtable 的跳变。

### 4.5 为什么跳变这么大

- `listpack`/`intset`：紧凑连续内存，**零指针开销**；
- `hashtable`/`skiplist`：每个元素一个节点（dictEntry/跳表节点，含前后指针），指针开销远大于数据本身；
- 阈值（config-thresholds.json）：**全部可 `CONFIG SET` 动态调**，是 DBA 平衡内存/性能的旋钮。

---

## 5. DBA 启示与编码速查

| 场景 | 现象 | 处置 |
| --- | --- | --- |
| Hash 卡在 512 字段附近 | 多塞 2 个字段内存 ×4.6 | 调 `hash-max-listpack-entries` 或拆分 key（OBS-003） |
| Set 全整数逼近 512 | 多 1 个整数 ×21.8 | 确认业务真的需要 intset；超限是突发内存尖峰 |
| 大 value（>64B）进集合 | 直接 hashtable | 压缩 value、或调 `*-max-listpack-value` |
| ZSet 范围查询为主 | listpack 无排序结构，转跳表才支持 | 命中 skiplist 是常态，不用怕 |
| 短 key + 短 value 海量 | kvobj 16B 头占比高 | 合并 key、用 Hash 打散（ENG-001） |
| `OBJECT ENCODING` 全表看 | 发现意外升级的 key | `MEMORY USAGE` 复核 + `--memkeys` 扫描（OBS-003） |

> 提醒：**编码升级是单向的**——listpack 转 hashtable 后不会自动转回（除非重建 key）。
> 所以"接近阈值"的业务要预判内存尖峰，而不是事后发现。

---

## 6. 与 PG 对照

| PostgreSQL | Redis |
| --- | --- |
| 行存储固定 layout + TOAST（大字段外置） | robj 头 + 编码切换（紧凑→通用） |
| 页面 8KB 对齐，tuple 头 ~24B | kvobj 16B 头，短数据内嵌 |
| TOAST 压缩大字段 | 字符串无压缩（RDB 落盘才压缩）；quicklist 节点可 LZF |
| 表膨胀需要 VACUUM | 编码升级即膨胀，只能重建 key |
| 索引结构固定（B-tree/哈希） | 同类型可换编码（intset↔HT、listpack↔skiplist） |
| `pg_column_size` 看字段大小 | `MEMORY USAGE` / `OBJECT ENCODING` 看对象大小 |

---

## 7. 小结

1. **robj 16 字节统一外壳**：type/encoding/refcount/lru + ptr，8.x 的 kvobj 把 key 也内嵌进来；
2. **String 三种编码**：int（零分配）/ embstr（一次 malloc）/ raw（两次 malloc）；8.x 阈值 = `key_len + value_len ≤ 41`；
3. **集合有"紧凑 → 通用"单向升级**：Hash 513 字段 ×4.6、Set 513 整数 ×21.8、ZSet 129 成员 ×12.5，List 无整体升级；
4. **阈值全部可动态调**：`CONFIG SET hash-max-listpack-entries` 等是 DBA 调内存的直接手段；
5. **8.x 变化**：共享整数不再进 keyspace（refcount=1）、embstr 阈值随键长变化——旧版知识要更新。
