# ENG-002-01 分布式锁与 Redis 原子能力

> Redis 8.6.2 / 127.0.0.1:6385 实测
>
> 缓存（ENG-001）解决了"读多写少"，分布式锁解决"多实例互斥"：多个应用节点抢同一个
> 临界区（发券、扣库存、幂等、定时任务唯一执行）。本文讲 Redis 怎么提供原子互斥，
> 以及 Redlock 的争议边界。证据：`evidence/lock-source.json`、`lock-experiment.json`。

---

## 0. 一句话心法

> **Redis 的锁 = 一条原子命令（SET NX EX）+ 一把"带 token 的 Lua 释放 + 过期兜底"。**
> 单线程事件循环让"检查+写入"天然原子（对比 PG 的锁要过 MVCC/事务）；
> 分布式锁的本质是**给分布式系统一个共享的互斥寄存点**，Redis 只是其中最常用的一种。
> 对 PG DBA：**SET NX EX ≈ pg_try_advisory_lock 的 Redis 版**，但语义有重要差异。

---

## 1. 为什么 Redis 能提供原子互斥

### 1.1 单线程 = 无并发窗口

`SET lock:order owner-A NX EX 30` 在 `setGenericCommand`（t_string.c:85）里一次完成
"检查不存在 → 写入 → 设过期"：

```text
SET key value NX EX 30
  NX: only if key doesn't exist（互斥判定）
  EX: 过期时间（防死锁兜底）
```

因为 Redis 单线程执行命令，这两件事之间**不可能插入其他客户端操作**——这就是原子性来源，
不需要 PG 那样的事务/行锁。对比反模式：

```text
SETNX lock:bad 1        # 命令 1：拿到锁
EXPIRE lock:bad 30      # 命令 2：设过期 —— 两步之间有崩溃窗口！
                       # 进程死在两步之间 → 锁永不释放
```

### 1.2 Lua 脚本 = 多步操作原子化

`EVAL` 里的多条 Redis 命令在事件循环内**同步、不可分割**执行（eval.c:552 luaCallFunction），
可以带逻辑分支。这比 MULTI/EXEC 强在"能写 if/else"，是锁释放/续期的关键。

---

## 2. 实验：完整锁流程

### 2.1 加锁（互斥）

```text
SET lock:order owner-A NX EX 30   → OK        # A 拿到锁
SET lock:order owner-B NX EX 30   → (nil)     # B 抢锁失败，立即返回不阻塞
```

> 与 PG 对比：`pg_advisory_lock` 拿不到锁会**阻塞等待**（或 `pg_try_advisory_lock` 立即返回 false）；
> Redis 的 SET NX 等价于 try-lock（立即返回），"等锁"逻辑要应用自己重试。

### 2.2 释放锁（必须校验 token）

直接 `DEL lock:order` 是**危险反模式**：如果 A 的锁已过期被 B 拿到，A 的 DEL 会误删 B 的锁。
正确做法是 Lua 校验 value：

```text
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1]
      then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:order owner-A
  token 不匹配 → 0（锁不动）
  token 匹配   → 1（删掉）
```

实测：用 `owner-B` 释放 `owner-A` 的锁返回 0，锁还在；用 `owner-A` 返回 1，锁消失。

### 2.3 过期兜底与续期

```text
SET lock:job worker-1 NX EX 3   # 持有者崩溃 → 3s 后自动释放，不会死锁
TTL lock:renew = 30             # 任务没做完怎么办？续期
EVAL "if GET==ARGV[1] then PEXPIRE KEYS[1] ARGV[2] else 0 end" 1 lock:renew worker-1 60000
  → TTL 30 → 60（续期成功）；attacker 续期 → 0（非持有者被拒）
```

这其实就是客户端库（Redisson 等）watchdog 的原型：后台线程周期性续期，
任务结束释放；持有者崩溃则过期自动让位。

---

## 3. 锁的边界：什么场景不需要锁

- **单命令原子**：INCR/DECR/SETNX 本身原子——实测 10 次并发 INCR = 10，无需加锁；
- 锁只保护**多步操作序列**（读→算→写），Redis 里可用 Lua 直接把整个序列原子化，
  很多时候"Lua 化"比"加锁"更优；
- 过期时间 vs 任务时长：过期太短任务没做完锁被抢（并发进入临界区）；
  太长持有者崩溃后锁长期占位——需要在"安全性"和"可用性"间权衡。

---

## 4. Redlock：共识争议

Redlock（antirez 提出）：对 **N 个互相独立的 Redis 节点**（建议 5）依次 SET NX EX，
拿到多数（N/2+1）且总耗时小于过期时间才算成功；释放时对所有节点删。

- **支持方（antirez）**：不依赖单点，容忍少数节点宕机；
- **质疑方（Martin Kleppmann）**：锁的过期/时钟跳跃/GC 停顿会让"两把锁同时生效"，
  Redlock 并没有比单节点更强的安全保证，正确性仍要靠 fencing token（分布式系统经典方案）；
- **DBA 视角结论**：
  - 单 Redis（主从+哨兵/集群）的 SET NX EX 对绝大多数业务**够用**；
  - 追求强安全（金融级）应接受"锁可能失效"，配合版本号/fencing token 二次校验；
  - Redlock 复杂度高、收益存疑，通常不必上。

---

## 5. 与 PG advisory lock 对照

| 维度 | Redis SET NX EX | PG pg_try_advisory_lock |
| --- | --- | --- |
| 获取方式 | 一条命令，立即返回 | try 版本立即返回；阻塞版等待 |
| 作用域 | key 存在即互斥（全实例） | 数据库内全局（按 bigint 键） |
| 过期兜底 | EX 自动过期（防死锁） | 无超时，靠会话结束释放 |
| 释放 | 持锁者 DEL（需 token 校验） | 会话结束/显式 unlock（事务级可自动） |
| 原子性来源 | 单线程命令 | 事务/锁管理器 |
| 适用 | 分布式多实例应用 | 单库多会话任务互斥 |

---

## 6. DBA 速查

- 加锁：`SET <lock-key> <token> NX EX <秒>`（token=UUID/随机串）；
- 释放：Lua 校验 token 后 DEL；**绝不用裸 DEL**；
- 续期：Lua 校验 token 后 PEXPIRE（watchdog 模式）；
- 多步操作优先 Lua 原子化，其次才考虑锁；
- 锁 key 的过期时间按"最坏任务时长"设置，配套续期机制；
- 监控：`INFO commandstats` 看 EVAL/SET 调用，锁 key 配合 MEMORY USAGE 看容量；
- 高可用：锁放主从/哨兵/集群 Redis；单点 Redis 挂了锁就没了（可用性），
  但"锁失效"通常比"服务不可用"好——按业务权衡。

---

## 7. 小结

- SET NX EX = 原子互斥原语，Lua = 多步原子化，两者构成分布式锁的完整工具链；
- 实测覆盖：互斥/防误删/过期兜底/续期/反模式，全部符合预期；
- Redlock 存在理论争议，生产优先"单点 Redis + token + 过期 + 续期"，必要时加 fencing token；
- 与 PG advisory lock 心智迁移：try-lock 语义相同，但 Redis 多一个"过期兜底"维度。
