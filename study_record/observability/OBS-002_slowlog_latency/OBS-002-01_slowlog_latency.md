# OBS-002-01 SLOWLOG 与延迟监控

> Redis 8.6.2 / 127.0.0.1:6379 / 密码 123456
>
> 本文回答两个问题：**"哪个命令慢？"（SLOWLOG）** 和 **"整个链路慢不慢？为什么慢？"（--latency / LATENCY）**。
> 所有数值均在本机实测（证据：`evidence/slowlog-entries.json`、`intrinsic-latency.json`、`latency-cli.json`、`latency-monitor.json`、`pg-mapping.json`）。

---

## 0. 一句话心法

> **SLOWLOG 告诉你"哪个命令慢"，`--latency` 告诉你"链路慢不慢"，LATENCY 告诉你"慢的瞬间发生了什么"——三层缺一不可。**
> 别一上来就怀疑 Redis：先分清是服务端慢、网络慢，还是客户端自己慢。

---

## 1. 三件套概览

| 工具 | 看什么 | 粒度 | 默认状态（本机实测） |
|---|---|---|---|
| `SLOWLOG` | 服务端哪个命令超时 | 单条命令，微秒级 | **开启**（本环境 >1ms 记录，环形保留 256 条） |
| `redis-cli --latency` | 客户端→Redis 往返延迟 | 端到端，毫秒级 | 随时可跑 |
| `redis-cli --intrinsic-latency` | 本机固有延迟（进程/内核） | 微秒级 | 随时可跑 |
| `LATENCY` 事件 | 延迟尖峰发生在哪类环节 | 事件+直方图 | 出厂默认关闭（0）；**本环境已开启**（100ms） |

> 注意：本实例（2026-08-25 重建后）`CONFIG` 与 `DEBUG` 均已启用、`LATENCY` 已开启，实验可直接做；**旧环境曾用 `rename-command` 禁用 CONFIG**——生产环境仍常见，遇到 `unknown command 'CONFIG'` 先怀疑安全加固，且 `SLOWLOG / LATENCY / --latency` 都不受影响。

---

## 2. SLOWLOG：先看配置，再读明细

### 2.1 配置（本机 redis.conf 实测）

```text
slowlog-log-slower-than 1000    # 本环境 1ms（出厂默认 10000us=10ms）
slowlog-max-len 256             # 本环境 256 条（出厂默认 128，超出丢最旧=环形缓冲）
```

- `CONFIG GET slowlog-log-slower-than` 可直接查（本环境已启用 CONFIG）；旧环境被禁用时看 `redis.conf`；
- 生产建议：`slowlog-log-slower-than` 设 1000~10000（1ms~10ms）之间，太大会漏掉关键慢命令，太小会被高频小慢命令刷屏。

### 2.2 制造几条真实慢命令（可复制执行）

```bash
# 方法一：Lua 纯计算脚本（不写 key、无副作用，最可控）
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning \
  EVAL "local s=0; for i=1,8000000 do s=s+i end return s" 0

# 方法二：真实数据结构 O(N) 操作（DB15 隔离库，做完 DEL 清理）
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning -n 15 \
  EVAL "local k=KEYS[1]; for i=1,300000 do redis.call('RPUSH',k,i) end return redis.call('LLEN',k)" 1 biglist
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning -n 15 \
  --raw SORT biglist LIMIT 0 1        # 全量排序只取 1 条 → 仍然慢
redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning -n 15 \
  LREM biglist 100000 150000          # 删除中间 10 万元素 → O(N) 拷贝
```

> 本环境 `DEBUG SLEEP` 已启用（`enable-debug-command local`，仅回环可用），可直接制造延迟；**EVAL 造慢命令的方法保留**——它不依赖 DEBUG，也是模拟真实 CPU 型慢命令的更干净手段。

### 2.3 SLOWLOG GET：逐字段解析（实测输出）

```text
127.0.0.1:6379> SLOWLOG GET 4
1) 1) (integer) 140          ← id：条目编号，从 0 起递增
   2) (integer) 1787644747   ← 时间戳（Unix 秒），换算北京时间 = 故障定位窗口
   3) (integer) 125107       ← 耗时（微秒），125107us ≈ 125ms
   4) 1) "EVAL"              ← 命令与完整参数（含 key 名）
      ...
   5) "127.0.0.1:43896"      ← 客户端 IP:端口，回溯哪个应用发的
2) ... id=139 LREM 22.5ms / 3) ... id=138 SORT 239ms / 4) ... id=137 EVAL 728ms
```

本机实测 4 条代表：

| id | 命令 | 耗时 | 说明了什么 |
|---|---|---|---|
| 137 | `EVAL` 脚本内循环 30 万次 `RPUSH` | 728 ms | Lua 脚本里批量写 = 一条"巨慢命令" |
| 138 | `SORT biglist LIMIT 0 1` | 239 ms | SORT 是全量排序，`LIMIT` 只限制返回，不限制排序 |
| 139 | `LREM biglist 100000 150000` | 22.5 ms | 大 list 中间删除 = O(N) 拷贝 |
| 140 | `EVAL` 800 万次纯 Lua 循环 | 125 ms | 纯计算也可能超阈值，CPU 型慢命令 |

### 2.4 LEN / RESET 与环形缓冲（实测）

```bash
SLOWLOG LEN        # 当前条数
SLOWLOG GET 1000000  # 取全部（最多仍只有 slowlog-max-len 条，本环境 256）
SLOWLOG RESET      # 清空列表
```

三个实测结论：

- **`GET N` 取的是"最近 N 条"，不是"历史第 N 条"**；传一个超大数（如 1000000）就是取全部；
- **环形缓冲**：连发 130 条 >10ms 慢命令后 `SLOWLOG LEN` 停在 128，最旧 id=9、最新 id=136，id 2~8 被挤出——超过 `slowlog-max-len` 丢最旧；
- **`RESET` 只清列表、不清 id 计数器**：实测 RESET 后新条目 id 从 137 继续递增，所以"id 有断号"是正常的。

---

## 3. 链路延迟：--latency 与 --intrinsic-latency

### 3.1 端到端往返（含网络）：`redis-cli --latency`

```bash
timeout 10 redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning --latency
# 实测输出（raw 格式，列 = min max avg samples，单位 ms）：
# 0 1 0.12 91
```

- 含义：6~10 秒内往返延迟 **min=0ms、max=1ms、avg≈0.12~0.17ms**（本机回环）；
- 输出列格式来自 `redis-cli.c` 的 `latencyModePrint`：`min max avg samples`；
- 用法：avg 正常但偶发 max 大 → 尖峰型问题，结合 SLOWLOG 看服务端；avg 持续高 → 网络/客户端问题，不一定是 Redis。

### 3.2 本机固有延迟：`redis-cli --intrinsic-latency <秒>`

```bash
timeout 15 redis-cli -h 127.0.0.1 -p 6379 -a 123456 --no-auth-warning --intrinsic-latency 3
```

实测输出（**注意参数单位是秒，不是次数**）：

```text
Max latency so far: 5 microseconds.
...
Max latency so far: 4831 microseconds.

584671 total runs (avg latency: 5.1311 microseconds / 5131.09 nanoseconds per run).
Worst run took 942x longer than the average latency.
```

- 平均 5.13us 极低，但偶发 4.8ms 尖峰（平均值的 942 倍）——这是"平均健康、偶发尖峰"的典型形态；
- 用途：判断"这台机器跑 Redis 本身快不快"。如果固有延迟都高（比如 >100us），先查内核调度/CPU 抢占/内存页错误/超卖，别急着找 Redis 的茬。

---

## 4. LATENCY 事件：慢的瞬间在哪个环节

### 4.1 出厂默认关闭（旧环境实测）

旧环境（重建前）未开启监控，实测：

```text
LATENCY DOCTOR
I'm sorry, Dave, I can't do that. Latency monitoring is disabled
in this Redis instance. You may use "CONFIG SET latency-monitor-threshold
<milliseconds>" in order to enable it.
LATENCY LATEST / HISTORY command → 空；LATENCY RESET → 0
```

### 4.2 本环境已开启（2026-08-25 重建后实测）

- 新实例 `redis.conf` 已写 `latency-monitor-threshold 100`（100ms）并启动，`CONFIG SET` 亦可用；
- 实测 `DEBUG SLEEP 0.3` 后：

```text
LATENCY DOCTOR → Dave, I have observed latency spikes...
1. command: 1 latency spikes (average 301ms, period 1.00 sec).
LATENCY LATEST → command  1787646595  301  301
```

- 结论：出厂默认 `0`=关闭（避免每命令判断开销）；生产按需开启，阈值建议 100~500ms，太低会被高频小尖峰刷屏；
- 开启后 `LATENCY DOCTOR` 会分类诊断，常见事件类别：
  `command`（命令超阈值）/ `fork`（RDB/AOF fork 耗时）/ `aof-write`（磁盘写）/ `eviction`（淘汰）/ `expire-cycle`（过期清理）；
- 8.x 另有命令级延迟跟踪（`latency-tracking yes` + `latency-tracking-info-percentiles`），同样需要 CONFIG 开启。

### 4.3 SLOWLOG vs LATENCY（新手最容易混）

| | SLOWLOG | LATENCY |
|---|---|---|
| 记录对象 | 每条超阈值命令的明细（谁/何时/什么命令/耗时） | 延迟事件的发生时间与采样直方图 |
| 关注点 | **定位到具体命令** | **看尖峰趋势与根因类别** |
| 默认 | 开启（10ms） | 出厂关闭（本环境已开 100ms） |
| 典型场景 | "下午 3 点哪条命令慢？" | "每天凌晨 fork 事件导致尖峰？" |

---

## 5. 延迟定位五步法（DBA 排障流程）

1. **先看链路**：`--latency` 确认端到端 avg/max；avg 正常 → 问题在服务端特定命令；avg 高 → 查网络/客户端；
2. **再看固有**：`--intrinsic-latency 5` 排除机器层（内核调度/超卖）；
3. **翻 SLOWLOG**：`SLOWLOG GET 100`，按耗时排序，找耗时最长的命令与 key；
4. **定位客户端**：用条目里的 `client addr` 反查是哪个应用/连接池（对应 `CLIENT LIST` 的 addr 字段）；
5. **开 LATENCY**：仍找不到就开启 `latency-monitor-threshold`，看尖峰落在 `command / fork / aof-write` 哪个类别，再对症处理（大 key 拆分、调 fork 频率、换 SSD 等）。

---

## 6. 与 PG 延迟监控对照

| Redis | PG | 实测对照结论 |
|---|---|---|
| `SLOWLOG`（默认 10ms 开） | `log_min_duration_statement`（默认 -1 关） | Redis 开箱即记慢命令；PG 默认不记，需手动开 |
| `SLOWLOG GET` 明细 | `pg_stat_statements`（mean_exec_time / max_exec_time） | 本机 PG 未装该扩展，启用需 `shared_preload_libraries='pg_stat_statements'`（重启）+ `CREATE EXTENSION` |
| `redis-cli --latency` | `pgbench -c1 -T10` 或 EXPLAIN ANALYZE | 端到端延迟 vs 单语句执行时间 |
| `LATENCY` 事件（command/fork/aof-write） | `pg_stat_activity.wait_event` / `pg_stat_bgwriter` | 等待事件分类定位根因，两边思路同构 |
| 三件套：SLOWLOG + LATENCY + --latency | 慢日志 + pg_stat_activity + 等待事件 | 监控体系可一一对应 |

---

## 7. 小结

- `SLOWLOG`：服务端慢命令明细，默认开启；会 `GET/LEN/RESET`，理解"最近 N 条"、128 环形截断、RESET 不清 id 计数器三个细节；
- `--latency` / `--intrinsic-latency`：链路与机器两层延迟体检，一个看端到端、一个看本机固有（实测 avg 5.13us、偶发 942 倍尖峰）；
- `LATENCY`：默认关闭，按需开启后按事件类别定位尖峰根因；
- 排障顺序：链路 → 机器 → 慢命令 → 客户端 → 事件，别一上来就怀疑 Redis。

**后续深化**：大 key / 热 key 发现与治理（SLOWLOG 里反复出现的 key 就是线索）进入 **OBS-003**。
