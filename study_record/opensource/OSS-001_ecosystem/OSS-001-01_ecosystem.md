# OSS-001-01 Redis 生态：Valkey 分支与云托管选型

> 2026-08 调研（本机基线 Redis 8.6.2；证据 `evidence/ecosystem-research.json`）
>
> 2024 年 3 月 Redis 换许可，Linux Foundation 拉 AWS/Google/Oracle 分叉出 Valkey——
> 这件事把"Redis"从一个软件名拆成了"一个品牌、两条分支、N 家云托管"。本文梳理分支
> 背景、许可差异、云厂商托管对比，给 DBA 一个选型框架。

---

## 0. 一句话心法

> **Valkey = Redis 7.2.4 的 BSD 分叉，协议完全兼容、云厂商主推；Redis 8 用 AGPLv3 补了
> 开源许可，但信任已裂。选型不用纠结"哪个更好"，按"许可合规 + 云托管归属 + 功能需求"
> 三条线定即可。**
> 对 PG DBA：这就是"PostgreSQL 永不分叉、而 MySQL 分叉出 MariaDB"的翻版——许可策略
> 决定生态走向，DBA 选型本质上是在选"治理主体"。

---

## 1. 事件线：为什么会有 Valkey

```text
2024-03  Redis Ltd. 将许可从 BSD → RSALv2/SSPLv1（云厂商做托管需商业协议）
2024-03-28  Linux Foundation 成立 Valkey 社区，从 Redis 7.2.4 fork，BSD-3-Clause
         AWS / Google / Oracle / Ericsson / Snap 牵头，Alibaba、Aiven、Percona 等跟进
2025-04-30  Redis 8.0 增加 AGPLv3 第三许可选项（OSI 批准，缓和社区情绪）
2025-04-19  Google Memorystore for Valkey GA
2025-12      腾讯云国内首发 Valkey 8.0（分叉后首个大版本）
```

关键点：Valkey 是**协议级兼容**的 fork——Redis 客户端不用改、配置不用改、RDB 能直接
载入、`REPLICAOF` 能在线同步，所以迁移成本极低；这也是它能被云厂商迅速接受的根本原因。

---

## 2. 许可对比（DBA 必须看的一栏）

| 维度 | Redis 8.x | Valkey 8.x |
| --- | --- | --- |
| 许可 | RSALv2 / SSPLv1 / AGPLv3 三选一 | BSD-3-Clause |
| 是否 OSI 开源许可 | 仅 AGPLv3 是；RSAL/SSPL 是"源代码可用" | 是（最宽松） |
| 云厂商托管 | 需与 Redis Ltd. 商业合作（Redis Cloud） | 无限制 |
| 再分发/商用 | AGPL 传染、RSAL/SSPL 有条件 | 完全自由 |
| 治理 | Redis Ltd.（公司主导） | Linux Foundation（社区主导） |

> 对 PG DBA 的类比：BSD ≈ PostgreSQL 许可（随便用随便改）；AGPL ≈ 加了"网络服务也
> 算分发"约束的开源许可；RSAL/SSPL ≈ "看得见源码但不是开源"。**给云上项目选引擎时，
> 先回答"谁托管、谁付费"，再看功能。**

---

## 3. 云托管对比（2026-08 现状）

| 云厂商 | Redis 系 | Valkey 系 | 备注 |
| --- | --- | --- | --- |
| AWS | ElastiCache for Redis OSS | **ElastiCache for Valkey**、**MemoryDB for Valkey**（持久化、多区域、向量检索，官方称成本 -30%） | 用 Valkey 8.1 替换 EOL 的 Redis 4.x/5.x |
| Google | Memorystore for Redis | **Memorystore for Valkey**（2025-04-19 GA，含托管向量检索） | Redis/Valkey/Memcached 三引擎 |
| Azure | Azure Cache for Redis（企业版=Redis Enterprise） | 官方尚无 Valkey 引擎 | 与 Redis 公司深度绑定 |
| 阿里云 | 云数据库 Redis 开源版（兼容 5.0/6.0/7.0） | **Tair 企业版**（Redis 兼容 + 扩展结构 + 持久内存/混合存储） | 国内 Redis 兼容生态另一极 |
| 腾讯云 | 云数据库 Redis | **Valkey 8.0 首发**（2025-12，社区贡献全球第一） | 国内最早支持 Valkey 8 |

> 结论一：AWS/GCP 已经"双轨制"（Redis + Valkey 都卖），新项目默认给 Valkey 引擎；
> Azure 仍押 Redis。结论二：阿里 Tair 走"协议兼容 + 自研扩展"路线（类似 PG 的
> Oracle 兼容分叉），与 Valkey 是两条不同的兼容路线。

---

## 4. 与 PG 生态对照

| 维度 | Redis 生态 | PG 生态 |
| --- | --- | --- |
| 许可风波 | 2024 Redis 换许可 → Valkey fork | PG 一直 BSD 类许可，无分叉 |
| 分叉 | Valkey（Linux Foundation） | MariaDB 之于 MySQL；PG 无主流分叉 |
| 云托管 | ElastiCache / Memorystore / Tair / Azure Cache | RDS for PostgreSQL / Cloud SQL / PolarDB / Aurora |
| 扩展生态 | Redis Stack 模块（JSON/Search/Bloom/Vector）逐步内建 | pgvector 等扩展进生态、部分进 core |
| 选型核心 | 许可合规 + 托管归属 + 功能缺口 | 版本特性 + 扩展可用性 + 托管归属 |

---

## 5. DBA 速查与选型建议

- 存量 Redis 应用迁 Valkey：**零代码**，RDB 导入或 `REPLICAOF` 在线切，先小流量灰度；
- 新项目默认考虑：云上托管（AWS/GCP）选 Valkey 引擎可规避许可/锁单风险；Redis 8 的
  AGPL 已可选，但"公司主导 + 曾改许可"的治理风险仍在；
- 需要 Redis 特有模块（RediSearch/RedisJSON 等 Stack 能力）时留意：Valkey 生态在
  Search/AI 方向追赶中（Valkey 8 已做相关演进），上线前按功能清单逐项验证；
- 许可红线：RSAL/SSPL 下"自己做托管卖给第三方"要商业协议；AGPL 下"对外提供网络服务"
  要考虑源码提供义务——**云上选型先问法务/采购，再问 DBA**；
- 国内场景：阿里 Tair / 腾讯 Valkey 8 都能覆盖 Redis 7.x 接口，兼容性以实际压测为准；
- 本学习仓库基线保持 Redis 8.6.2（开源版、可本地复现）；后续若补 Valkey 对照实验，
  建议用 docker 起 Valkey 8 与 6385 实例做协议/性能对比（SUNION/SDIFF 等）。

---

## 6. 小结

- Valkey 是许可事件催生的 BSD fork，协议兼容 + Linux Foundation 治理 + 云厂商主推，
  已事实上成为"开源 Redis"的新默认；
- Redis 8 用 AGPLv3 补开源许可，但 RSAL/SSPL 仍在，托管/再分发需注意条款；
- 云托管：AWS/GCP 双轨、Azure 押 Redis、国内 Tair/Valkey 各有生态，选型看"托管归属 +
  许可 + 功能缺口"；
- 与 PG 对照：Redis 经历了 MySQL→MariaDB 式的分叉，而"扩展进核心"的路线与 PG 同构。
