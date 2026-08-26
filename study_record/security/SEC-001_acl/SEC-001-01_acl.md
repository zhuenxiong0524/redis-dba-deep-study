# SEC-001-01 ACL 权限体系与安全加固

> Redis 8.6.2 / 独立实例 127.0.0.1:6385（aclfile 持久化，rename-command 启用）
>
> 前面所有实验都用 `requirepass` 或裸奔；本文把**权限体系**讲透：
> 用户怎么建、命令/key/通道怎么授权、越权怎么审计、危险命令怎么禁、密码怎么存。
> 全部本机实测，证据：`evidence/acl-source.json`、`acl-users-experiment.json`、`acl-ops-experiment.json`。

---

## 0. 一句话心法

> **ACL = 用户（密码）+ 命令权限 + Key 权限 + 通道权限 四件套；requirepass 只是 default 用户的快捷方式。**
> 对 PG DBA：**ACL 用户 ≈ PG role，命令权限 ≈ GRANT/REVOKE 的命令级版本，
> Key 权限 ≈ PG 没有内置的行级权限（最接近的是 RLS），通道权限 ≈ PG 的 PUBSUB 没有对应物。**

---

## 1. 模型：一个 user 结构

源码 `user`（server.h:1328）：

```text
user { name, flags, passwords[], selectors[], acl_string }
  flags:  ENABLED(on) / DISABLED(off) / NOPASS / SANITIZE_PAYLOAD
  passwords: SHA256 hex 列表（ACL GETUSER 显示，AUTH 时比对）
  selectors: 权限选择器（命令 bitmap + key 模式 + 通道模式）
      SELECTOR_FLAG_ALLKEYS / ALLCOMMANDS / ALLCHANNELS
```

默认用户：

```text
user default on nopass sanitize-payload ~* &* +@all
           │    │      │                 │  │   └── 全部命令
           │    │      │                 │  └────── 全部通道 &*
           │    │      │                 └───────── 全部 key ~*
           │    │      └── RESTORE 深校验载荷
           │    └── 无需密码（本实例未配置）
           └── 启用
```

规则语法（`ACL SETUSER`，解析在 acl.c:1284 `ACLSetUser`）：

| 规则 | 含义 |
| --- | --- |
| `on` / `off` | 启用 / 禁用用户 |
| `>password` / `<password` | 加 / 删明文密码（存 SHA256） |
| `#hash` | 直接给 SHA256 哈希（不传明文） |
| `nopass` / `resetpass` | 免密 / 清空密码 |
| `~pattern` / `%R~pattern` / `%W~pattern` | key 全部 / 只读 / 只写模式 |
| `resetkeys` / `allkeys` | 清空 key 模式 / 允许所有 key |
| `+cmd` / `-cmd` | 允许 / 禁止某命令（`config|get` 子命令级） |
| `+@category` / `-@category` | 允许 / 禁止某命令类别（`ACL CAT` 可查） |
| `&channel` / `resetchannels` / `allchannels` | 通道白名单（pub/sub） |

---

## 2. 实验：用户权限矩阵

### 2.1 alice：key 白名单 + 禁危险命令

```text
ACL SETUSER alice on >alice-pass ~cache:* +@all -@dangerous
```

| 操作 | 结果 |
| --- | --- |
| `SET cache:user:1 v1` / `GET` | OK / v1 |
| `SET other:key v` | `NOPERM No permissions to access a key`（key 越权） |
| `KEYS *` | `NOPERM ... 'keys' command`（@dangerous） |
| `CONFIG GET maxmemory` | `NOPERM ... 'config|get' command`（子命令级！） |
| `AUTH alice wrong-pass` | `WRONGPASS` |

关键点：**权限粒度细到子命令**（`config|get` 与 `config|set` 分开判定）；
`@dangerous` 包含 KEYS/FLUSHALL/MONITOR/SHUTDOWN/SORT 等阻塞或破坏性命令。

### 2.2 bob：最小命令集

```text
ACL SETUSER bob on >bob-pass ~* +get +set     # 默认 -@all，只加两条
SET k1 v1 → OK；GET k1 → v1；DEL k1 → NOPERM
```

### 2.3 dave：Key 只读模式（Redis 7+）

```text
ACL SETUSER dave on >d-pass resetkeys %R~cache:ro:* +@all -@dangerous
SET cache:ro:a 1 → NOPERM        # 只读模式拒绝写
GET cache:ro:a   → 100           # 可读
```

> 坑：`~*`（allkeys）之后再追加 `%R~...` 会报错"Adding a pattern after the * pattern is not valid"，
> 必须先 `resetkeys`。设计上 allkeys 是"封顶"授权，不能再细化。

### 2.4 eve：pub/sub 通道白名单

```text
ACL SETUSER eve on >e-pass ~* &chat:* +@all
SUBSCRIBE chat:news   → subscribe OK
SUBSCRIBE other:news  → NOPERM No permissions to access a channel
```

### 2.5 禁用用户

```text
ACL SETUSER carol off 后 AUTH carol c-pass → WRONGPASS ... or user is disabled.
```

---

## 3. 实验：审计与运维能力

### 3.1 ACL LOG：越权全记录

刚才所有 NOPERM/WRONGPASS 都被记录：

```text
ACL LOG
1) reason=command  object=keys        username=alice
2) reason=command  object=config|get  username=alice
3) reason=key      object=other:key   username=alice
4) reason=auth     object=AUTH        username=alice
5) reason=command  object=del         username=bob
6) reason=key      object=cache:ro:a  username=dave
7) reason=channel  object=other:news  username=eve
```

字段：`reason(command/key/auth/channel)` + `object` + `username` + 时间戳 + 客户端信息。
用途：**排查"谁在尝试越权"**，对应 PG 的 failed login / 审计日志。默认最多保留 128 条。

### 3.2 requirepass 只是 default 用户的快捷方式

```text
CONFIG SET requirepass 123456
→ 未认证 PING 报 NOAUTH
→ AUTH 123456 后 OK
→ ACL GETUSER default 的 passwords = 8d969eef...（正是 sha256("123456")）
```

所以老教程里的 `requirepass` 心智 = 只给 default 用户设密码；**ACL 是它的超集**。

### 3.3 ACL SAVE / aclfile：权限持久化

```text
ACL SAVE → 写 users.acl（配置 aclfile 指定路径）
user alice on sanitize-payload #9b90e524... ~cache:* resetchannels +@all -@dangerous
```

重启后 `ACL LIST` 完整恢复、`AUTH alice alice-pass` 可用——**权限独立于数据持久化**。

### 3.4 rename-command：危险命令改名/禁用

```text
rename-command CONFIG ""                  # 彻底禁用
rename-command FLUSHALL redis_flushall_8x # 改名
rename-command KEYS ""                    # 彻底禁用
```

| 操作 | 结果 |
| --- | --- |
| `CONFIG GET maxmemory` | `ERR unknown command 'CONFIG'` |
| `KEYS *` | `ERR unknown command 'KEYS'` |
| `redis_flushall_8x` | OK（改名生效） |
| alice 执行 `redis_flushall_8x` | `NOPERM ... 'flushall' command` |

要点：
- **运行期 `CONFIG SET rename-command` 报错**，只能改配置文件重启（与 ACL SETUSER 不同）；
- **ACL 权限按原命令名判定**——改名后 alice 的 `-@dangerous` 依然拦住 flushall；
- 这是对 ARC-003（阻塞命令）/CLU 系列危险面的最后一道闸。

---

## 4. 与 PG 权限对照

| Redis | PostgreSQL | 说明 |
| --- | --- | --- |
| ACL 用户 + 密码(SHA256) | role + password（pg_authid 存哈希） | 认证模型 |
| requirepass | postgres 超级用户密码 / pg_hba | 单密码快捷方式 |
| `+cmd` / `-cmd` 命令权限 | GRANT EXECUTE / REVOKE | Redis 粒度到子命令 |
| `~pattern` key 权限 | 表/列级 GRANT；行级需 RLS | PG 默认无行级，RLS 是近似物 |
| `%R~` / `%W~` 读写模式 | SELECT / INSERT-UPDATE-DELETE 分离 | 语义相近 |
| `&channel` 通道权限 | 无对应（PG 无内建 pub/sub 权限） | Redis 特色 |
| ACL LOG | pg_stat_activity + log_connections | 审计视角 |
| rename-command | 无直接对应（可 REVOKE 或 pg_hba） | 禁用面 |

---

## 5. DBA 速查

- 建用户最小权限：`ACL SETUSER app on >pass ~app:* +@all -@dangerous -@admin`；
- 生产不裸奔：default 用户 `ACL SETUSER default off` 或设密码，业务全部走命名用户；
- 查权限：`ACL GETUSER <user>` / `ACL LIST` / `ACL CAT`；生成密码：`ACL GENPASS`；
- 审计：`ACL LOG`（越权追踪），配合 `CONFIG SET acllog-max-len`；
- 持久化：配 `aclfile`，变更后 `ACL SAVE`，重启自动加载；
- 危险命令：优先 ACL `-@dangerous`，再考虑 rename-command（KEYS/CONFIG/FLUSHALL/MONITOR/SHUTDOWN/SORT）；
- 密码轮换：`>new` 后 `resetpass` 旧密码即刻生效；禁用账号：`off`；
- 复制/哨兵/集群场景：从库 `masteruser`/`masterauth` 配 ACL 用户，主从认证用 `user:pass`。

---

## 6. 小结

- 四维权限：用户 × 命令（到子命令）× Key（到读写模式）× 通道，粒度远超 PG 默认模型；
- requirepass 只是 default 用户密码的别名；ACL 是唯一现代做法；
- ACL LOG 是安全审计的抓手，ACL SAVE 让权限像配置一样可版本化；
- rename-command 是最后防线，注意它"按原命令名判权限"的细节；
- 下一步：ENG-002 分布式锁（SETNX+Lua 原子能力），或 OSS-001 生态对比（Valkey/云托管）。
