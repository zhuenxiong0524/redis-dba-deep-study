# 事务、管道与 Lua 脚本

> Redis 8.6.2 / Debian 12 / 127.0.0.1:6379
>
> 从 PG 迁移的 DBA 最容易在"事务"上踩坑：Redis 的 `MULTI/EXEC` 和 PG 事务**名字像、语义完全不同**。
> 本任务用实测讲清三件事：MULTI/EXEC 到底保证什么、WATCH 乐观锁怎么用、pipeline 为什么快、
> Lua 脚本为什么是"真正的原子操作"。

---

## 1. 先理解前提：Redis 单线程执行

Redis 的命令由**单线程事件循环**顺序执行（见 ARC-001）。这个前提带来两个推论：

1. **单条命令天然原子**：`INCR` 不会读到半截值，不需要锁；
2. **"事务"的意义变了**：PG 事务解决"并发冲突 + 回滚"，Redis 根本不存在并发执行，
   所以它的事务只需要解决一件事——**让一批命令不被其他客户端"插队"**。

> 一句话：PG 事务 = 隔离 + 回滚；Redis MULTI/EXEC = **原子排队**（无回滚、无隔离级别）。

---

## 2. MULTI / EXEC / DISCARD：打包执行

### 2.1 正常流程（实测）

```text
MULTI            # OK（进入事务队列模式）
SET k1 v1        # QUEUED（不执行，先排队）
INCR k2          # QUEUED
EXEC             # 1) OK  2) 1（依次执行，返回所有结果）
```

- 从 `MULTI` 到 `EXEC` 之间的命令**只排队不执行**（返回 `QUEUED`）；
- `EXEC` 一次性按序执行；**执行期间其他客户端命令不会插入**（单线程 + 排队即原子）。

### 2.2 队列期错误 → 整个事务放弃（实测）

```text
MULTI
SET k1 x
WRONGCMD          # ERR unknown command 'WRONGCMD'（排队时就报错）
SET k3 y
EXEC              # EXECABORT Transaction discarded because of previous errors
```

结论：**命令拼写/语法错误在排队时就能发现**，`EXEC` 直接放弃，k1/k3 都没执行。

### 2.3 运行期错误 → 无回滚（实测，和 PG 最大的区别）

```text
SET k2 abc                    # 先造一个非数字值
MULTI
SET k1 new1                   # QUEUED
INCR k2                       # QUEUED（此时还不知道会错）
SET k3 new3                   # QUEUED
EXEC                          # 1) OK  2) ERR value is not an integer  3) OK
最终：k1=new1（执行了） k2=abc（INCR 失败，值没变） k3=new3（执行了）
```

结论：**某条命令运行期报错，不影响其他命令执行，也不会回滚已执行的**。
PG 的 `ROLLBACK` 能撤销一切；Redis 没有这个概念——写错了只能自己补偿。

### 2.4 DISCARD：丢弃排队命令

```text
MULTI
SET k1 discard    # QUEUED
DISCARD           # OK（丢弃排队命令，退出事务模式）
```

> 注意：`DISCARD` 只是"不要这批命令了"，**不是回滚**（还没执行，谈不上回滚）。

---

## 3. WATCH：乐观锁（双客户端实测）

`MULTI/EXEC` 只能防"插队"，不能防"你读到的值已经过期"。WATCH 解决后者：
**监视 key，如果在你 WATCH 之后、EXEC 之前被其他客户端修改，EXEC 返回 nil（放弃）**。

### 3.1 场景 A：WATCH 后他人修改 → EXEC 放弃

```text
客户端1: WATCH counter            # OK（开始监视）
客户端2: INCR counter             # 10 → 11（在客户端1事务前修改了 counter）
客户端1: MULTI
         SET counter 999          # QUEUED
         EXEC                     # (nil)  ← 放弃！999 没有生效
最终: counter = 11
```

### 3.2 场景 B：WATCH 后无人修改 → 正常提交

```text
客户端1: WATCH counter            # OK
客户端1: MULTI / SET counter 999 / EXEC
最终: counter = 999               # EXEC 返回 OK，提交成功
```

### 3.3 场景 C：UNWATCH 解除监视

```text
WATCH counter
UNWATCH                          # 解除所有监视
MULTI / SET counter 2 / EXEC     # 正常执行
```

### 3.4 与 PG 对照

| PG | Redis |
| --- | --- |
| `SELECT ... FOR UPDATE`（悲观锁） | 无（单线程不需要行锁） |
| `SELECT` 旧值 → 条件 `UPDATE ... WHERE 条件` | `WATCH` + `MULTI` + `EXEC` |
| 冲突 → 锁等待/死锁检测 | 冲突 → EXEC 返回 nil，**客户端重试** |
| 适合写多 | 适合读多写少（乐观锁，冲突重试成本低） |

> 经典用法（库存扣减/抢购）：`WATCH stock` → `GET stock` → 业务判断 → `MULTI` 里 `DECR` → `EXEC`；
> 返回 nil 就重读重试。更简单可靠的是 Lua（见 §5.3）。

---

## 4. pipeline：为什么快 155 倍（实测）

### 4.1 三种方式跑 2000 条 SET

| 方式 | 实现 | 耗时 | 说明 |
| --- | --- | --- | --- |
| 顺序执行 | 2000 次独立连接逐个 SET | **4.64s** | 每次都有网络往返（RTT） |
| `redis-cli --pipe` | 命令一次性批量发送 | **0.03s** | 1 次往返（约 155 倍） |
| 单个 MULTI/EXEC | 2000 条包在一个事务 | **0.11s** | 1 次往返（约 42 倍） |

### 4.2 原理与区别

```text
顺序：SET → 等回复 → SET → 等回复 → ...   # 慢在网络往返，不是 Redis 执行慢
pipeline：SET SET SET ... → 一次收回复    # 批量发送，客户端攒批
MULTI/EXEC：命令排队，EXEC 一次性执行      # 既有批量又有原子性
```

| | pipeline | MULTI/EXEC |
| --- | --- | --- |
| 批量省 RTT | ✅ | ✅ |
| 原子性（不被插队） | ❌ | ✅ |
| 中途错误 | 各自独立 | 见 §2.3（无回滚） |
| 用途 | 批量灌数据/读多 key | 需要"一批必须连着执行" |

### 4.3 DBA 建议

- 客户端库的 pipeline 是**自动攒批**的（如 jedis `pipeline`、redis-py `pipeline`）；
- 单批别贪大：100~1000 条一批为宜，避免单次网络包过大/内存尖峰；
- 大批量初始化用 `redis-cli --pipe`（工具本身自带批量协议）。

---

## 5. Lua 脚本：Redis 的"存储过程"

### 5.1 基础：EVAL

```text
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 dat004:lua:key hello   # OK
EVAL "return redis.call('GET', KEYS[1])" 1 dat004:lua:key                  # hello
```

- `KEYS[1]` 传 key（必须走 KEYS，便于集群路由），`ARGV[1]` 传参数；
- 脚本内可以写 `if/else`、循环——这是 MULTI/EXEC 做不到的**逻辑能力**。

### 5.2 原子性（实测）

```text
EVAL "SET key '1'; INCR key; INCR key; return GET key" 1 dat004:lua:key   # 3
```

脚本执行期间，其他客户端命令**不会插入**——所以脚本 = "带逻辑的原子事务"。

### 5.3 CAS（Compare And Swap）脚本：WATCH 的升级版

```text
-- 只有当 key 当前值等于期望值时，才更新（防并发覆盖）
EVAL "local v = redis.call('GET', KEYS[1]);
      if v == ARGV[1] then redis.call('SET', KEYS[1], ARGV[2]); return 1
      else return 0 end" 1 dat004:lua:stock 5 8

实测：stock=5，传旧值 5 新值 8 → 返回 1，stock=8
      再传旧值 5（实际已是 8）→ 返回 0，stock 保持 8
```

> 这就是**分布式锁、秒杀扣库存、防重复提交**的底层实现（ENG-002 深入）。比 WATCH+MULTI
> 更简单：比较和修改在脚本内完成，天然原子，不用重试循环。

### 5.4 脚本错误语义

```text
EVAL "SET key 'x'; INCR key" 1 k   # SET 已生效，INCR 报错中止脚本
```

和 MULTI/EXEC 一样**无回滚**：错误前的写入保留。所以脚本里要自己做好检查
（先判断类型/存在性再操作）。

### 5.5 工程化：EVALSHA 与 Functions

- `EVALSHA`：先用 `SCRIPT LOAD` 把脚本缓存，之后用 sha 调用，省带宽（脚本体不用每次传）；
- Redis 7+ **Functions**（`FUNCTION LOAD` + `FCALL`）：给脚本起名、集中管理、可版本化，
  是"存储过程"的正式形态，推荐新项目使用：

```text
FUNCTION LOAD "#!lua name=mylib
redis.register_function('setget', function(keys, args)
  redis.call('SET', keys[1], args[1])
  return redis.call('GET', keys[1])
end)"
FCALL setget 1 mykey myvalue    # 调用注册的函数
```

---

## 6. 选型速查：什么时候用哪个

| 需求 | 用什么 |
| --- | --- |
| 只是批量执行、不要求原子 | **pipeline**（快） |
| 一批命令必须连着执行（不被插队） | **MULTI/EXEC** |
| 读-判断-写 需要"比较后更新" | **Lua 脚本**（首选）或 WATCH+MULTI |
| 需要 if/循环等逻辑 | **Lua**（MULTI 做不到） |
| 可复用/正式管理 | **Functions**（Redis 7+） |
| 需要回滚（PG 语义） | **Redis 没有**，靠业务补偿/幂等设计 |

---

## 7. 与 PG 事务对照表

| PostgreSQL | Redis |
| --- | --- |
| `BEGIN/COMMIT` | `MULTI/EXEC` |
| `ROLLBACK`（回滚全部） | 无回滚；`DISCARD` 只丢排队命令 |
| 隔离级别（READ COMMITTED 等） | 无（单线程，天然串行） |
| 悲观锁 `SELECT FOR UPDATE` | 无（不需要） |
| 乐观锁（条件 UPDATE / 版本号） | `WATCH` + `EXEC`（冲突返回 nil） |
| 存储过程 PL/pgSQL | Lua 脚本 / Functions |
| 批量操作 | pipeline / `redis-cli --pipe` |
| 并发控制目标 | 防止"读旧写新"（靠脚本/WATCH） |

**核心差异一句话**：PG 事务保护"多行数据的一致性"，失败可回滚；Redis 事务/脚本保护
"几个命令的原子性"，失败不回溯——设计业务时要把"要么全成、要么全败"的期望
放进 Lua 脚本或应用层补偿里。

## 8. 路径与证据

```text
任务目录  study_record/datatype/DAT-004_transaction_pipeline_lua/
证据      evidence/multi-exec.json         (MULTI/EXEC/DISCARD 与两类错误)
          evidence/watch-optimistic.json   (WATCH 三场景双客户端实测)
          evidence/pipeline-performance.json (2000 SET 三方式耗时)
          evidence/lua-script.json         (EVAL/CAS/错误/Functions/EVALSHA)
源码      /data/redis_pkg/redis-stable/src/multi.c / script.c / eval.c / functions.c
```
