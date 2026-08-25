# ARC-001-01 单线程事件循环与 I/O 多路复用

> Redis 8.6.2 / 127.0.0.1:6379
>
> 这是 Redis 架构系列第一篇：回答 DBA 听到最多的那个问题——**"Redis 单线程为什么还这么快？"**
> 并且用源码路径 + 实测证明"单线程"的确切含义和边界。
> 实测证据：`evidence/event-loop-source.json`、`single-thread-blocking.json`、`thread-structure.json`、
> `io-threads-experiment.json`、`event-loop-metrics.json`。

---

## 0. 一句话心法

> **Redis 的"单线程"指的是：命令执行只有一条线程（主线程），一条命令执行完才执行下一条。**
> 但"等待网络/磁盘"这类 I/O 不占这条线程——epoll 帮它一个人盯着成千上万个连接。
> 所以：**快是因为不用锁，怕的是慢命令——一个慢命令 = 所有人排队。**

---

## 1. "单线程"到底指什么：进程线程结构实测

先破除误解。`io-threads=1` 时（本机默认），进程**不是只有 1 个线程**，实测 `/proc/<pid>/status`：

```text
Threads:	6
redis-server       ← 主线程：事件循环 + 命令执行（唯一的执行线程）
bio_close_file     ← 后台线程：异步关闭 fd
bio_aof            ← 后台线程：异步 AOF fsync
bio_lazy_free      ← 后台线程：异步释放内存（UNLINK 等）
jemalloc_bg_thd ×2 ← 内存分配器后台线程
```

结论：

- **执行命令的只有主线程 1 条**：`INFO threads` 只有 `io_thread_0`，`io_threaded_reads_processed=0`；
- 另外 3 个 `bio_*` 是"跑腿"线程，只做异步收尾（关连接、刷盘、延迟释放大对象），**不碰命令执行**；
- 所以 OBS-003 实测的 `UNLINK` 比 `DEL` 快（155ms vs 39ms），靠的就是 `bio_lazy_free` 异步释放。

---

## 2. 事件循环：ae.c + epoll（源码路径）

源码：`/data/redis_pkg/redis-stable/src`，链路如下：

```text
main()                                   server.c:7683
 └─ initServer()                          server.c:2862
     └─ aeCreateEventLoop()               server.c:2937  创建事件循环
     └─ createSocketAcceptHandler()       server.c:2706  监听 socket 注册 accept 事件
 └─ aeMain(server.el)                     server.c:8027  ← 进入死循环，永不返回
     └─ aeProcessEvents()                 ae.c:360
         ├─ beforeSleep()                 server.c:1857  写回复/刷AOF/快速过期
         ├─ aeApiPoll() → epoll_wait()    ae_epoll.c:89  ← 阻塞等待事件（唯一"睡"的地方）
         └─ 按 fd 分发 rfileProc/wfileProc ae.c:444       ← 就绪连接逐个处理
```

关键点：

- **一个进程只有一个 epoll fd**（`epoll_create(1024)`，ae_epoll.c:19），所有连接都注册在上面；
- `epoll_wait` 一次性返回所有就绪 fd，主线程**批量**处理，处理完继续 `epoll_wait`；
- `INFO server` 的 `multiplexing_api:epoll` 就是 `aeGetApiName()` 返回的当前实现；
- 每轮循环结束前 `beforeSleep` 统一把客户端输出缓冲写回（合并小包、减少系统调用）。

> 与 PG 对照：PG 是"一连接一进程/线程，各自阻塞在 read()"；Redis 是"一个线程阻塞在 epoll_wait()，
> 谁有数据就处理谁"。前者并发靠堆进程，后者并发靠 I/O 多路复用。

---

## 3. 一条命令的一生（命令处理主流程）

```text
epoll 就绪 → readQueryFromClient()            networking.c:3715   read() 收数据入 querybuf
          → processInputBuffer()              networking.c:3529   解析 RESP 协议成命令
          → processCommandAndResetClient()    networking.c:3394   设置 current_client
          → processCommand()                  server.c:4297       权限/参数/maxmemory/集群检查
          → call()                            server.c:3830
              └─ c->cmd->proc(c)              真正执行（如 getCommand）
          → beforeSleep 中写回回复             server.c:1857
```

要点：

- 命令在 `call()` 里执行（server.c:3830），执行完就地统计耗时 → 慢日志、`commandstats`、LATENCY；
- 回复不立即写回，而是攒在输出缓冲，**每轮循环的 `beforeSleep` 统一 flush**——这就是高吞吐的原因之一；
- 整个链路里**没有锁、没有上下文切换**：这就是单线程快（低延迟、高吞吐）的根本。

---

## 4. 单线程的代价：一个慢命令拖垮所有客户端（实测）

实验：客户端 A 执行 `DEBUG SLEEP 3`（睡 3 秒），客户端 B 同时采样 PING 延迟：

```text
客户端 A：DEBUG SLEEP 3 → OK（3 秒后才返回）
客户端 B：--latency 6s 采样 90 次 → min=0ms, avg=33.42ms, MAX=2998ms
SLOWLOG：DEBUG SLEEP 3  duration=3000614µs（≈3.0s）
cmdstat_debug: usec_per_call=3002372.50（平均 3 秒/次）
cmdstat_ping:  usec_per_call=1.03
```

结论（这就是"串行"的实锤）：

- 慢命令执行期间，**所有其他客户端的任何命令都排队**，PING 这种 1µs 的命令也被拖到 3 秒；
- 只要有一条慢命令，事件循环就被卡住：连接不被读取、回复不被写回、过期检查/AOF 也暂停；
- DBA 第一戒律：**禁止在生产执行 KEYS/SMEMBERS/大 DEL/大 SORT**（详见 ARC-003）。

---

## 5. 为什么单线程还能扛几十万 QPS

单线程不慢，前提是命令本身快：

- 实测 50 并发 × 20 万请求：`PING_INLINE ≈ 37,994 rps`（p50=0.695ms）、`PING_MBULK ≈ 37,092 rps`（p50=0.687ms）；
- 事件循环基础开销 `--intrinsic-latency`：**平均 4.97µs/轮**（本进程内，无网络）；
- 小命令 O(1)/O(log N)、无锁、无上下文切换、回复批量写回 → 单线程足够；
- pipeline 下还能更高：DAT-004 实测 pipeline 2000 条命令提速 **155x**（epoll 批量 + 免等待）。

瓶颈在哪：**网络往返 + 内存 + 慢命令**，而不是 CPU 单线程本身（本 VM 仅 1 核也能跑 3.7 万 rps）。

---

## 6. io-threads：并行的是 I/O，不是命令执行

Redis 6.0 引入 `io-threads`，8.x 已成熟。先看源码边界：

```text
initThreadedIO()  iothread.c:868   if (server.io_threads_num <= 1) return;   ← ≤1 直接不建线程
IOThreadMain()    iothread.c:854   每个 IO 线程跑自己的 aeMain 事件循环
                                   ——只做 socket 读/写 + 解析，不执行命令
config.c:3208     io-threads 为 IMMUTABLE_CONFIG（不支持 CONFIG SET，改需重启）
config.c:455      io-threads-do-reads 已废弃（8.x 中 >1 时读写都走 IO 线程）
```

实测对照（临时实例 6390，50 并发 × 30 万 PING，本机 1 核）：

| 配置 | 进程线程数 | PING_INLINE | PING_MBULK | io_threaded_reads | io_threaded_writes |
| --- | --- | --- | --- | --- | --- |
| `io-threads 1` | 6 | **40,546 rps** | **39,725 rps** | 0 | 0 |
| `io-threads 4` | 9 | 21,831 rps | 22,497 rps | 600,103 | 600,001 |

- `io_threaded_*` 计数证实：开 4 线程后读写**确实被 IO 线程分担**（`INFO threads` 显示 io_thread_1/2/3 各 ~20 万次读写）；
- 但命令执行仍只有主线程，且**线程间交接（pending 队列 + 事件通知）有额外开销**；
- 本机只有 1 核：无并行收益、只有交接成本 → `io-threads=4` 反而慢约 **46%**；
- 适用场景：**多核 + 网络 I/O 是瓶颈**（大 value 读写、TLS、大量短连接）。小命令密集场景别开。

---

## 7. DBA 启示与阻塞速查

| 情形 | 影响 | 处置 |
| --- | --- | --- |
| 慢命令（KEYS/SMEMBERS/大 DEL/SORT） | 全体客户端排队（实测 3s） | `SCAN` 分批、`UNLINK`、禁 KEYS；SLOWLOG 盯 `duration` |
| 大 value 读写 | 单命令耗时长 + 网络传输慢 | 拆分/压缩（OBS-003） |
| fork（BGSAVE/AOF 重写） | 事件循环暂停瞬间（COW 拷贝页表） | PER-001：`latency-monitor-threshold` 盯 fork 事件 |
| 客户端数巨大 | epoll 事件处理增多，单轮变长 | 连接池、`maxclients`、监控 `INFO clients` |
| 单核 + 开 io-threads | 更慢（实测 -46%） | 按 CPU 核数决定，先压测验证 |

---

## 8. 与 PG 对照

| PostgreSQL | Redis |
| --- | --- |
| 一连接一后端进程，各自 read() 阻塞 | 单线程 epoll 多路复用，一个线程盯所有连接 |
| 并发模型：进程数 = 连接数（有上限） | 并发模型：事件循环 + 回调（无进程开销） |
| 慢 SQL 只拖累自己会话 | **慢命令拖累所有人**（全局串行） |
| `pg_stat_activity` 看会话状态 | `CLIENT LIST` + `INFO commandstats` + SLOWLOG |
| autovacuum/checkpoint 后台进程 | `bio_*` 后台线程 + fork 子进程（BGSAVE） |
| 多核跑并行查询（parallel workers） | 命令执行固定单线程；io-threads 只并行 I/O |
| 无 MVCC、无锁（单线程天然串行） | 命令原子，天然无竞态（对比 DAT-004 事务语义） |

---

## 9. 小结

1. **单线程 = 命令执行串行**（主线程 1 条），bio_* 后台线程只做异步收尾；
2. **快的原因**：epoll 多路复用 + 无锁 + 回复批量写回（`beforeSleep`）；
3. **慢命令是全局灾难**：`DEBUG SLEEP 3` → 其他客户端延迟 2998ms，SLOWLOG 记录 3.0s；
4. **io-threads 并行的是 I/O 不是执行**：1 核 VM 实测开 4 线程慢 46%，多核网络瓶颈场景才受益；
5. **DBA 心法**：把慢命令消灭在源头，单线程就永远不是瓶颈。
