# Redis DBA 一年学习项目

面向 PostgreSQL / Oracle DBA 的 Redis 体系化学习仓库，任务基线 **Redis 8.6.2**。

- 6 周快速路线图（含 PG ↔ Redis 概念对照表）：`redis-learning-roadmap.md`
- 全年任务全集与权重体系：`study_record/learning-roadmap.md`
- 按专题沉淀文章 + 证据（evidence/），仿照 PG 学习仓库 `postgresql-dba-deep-study` 的结构

## 目录结构

```text
redis-learning-roadmap.md   6 周学习路线图（PG/Oracle DBA 视角）
study_record/
├── learning-roadmap.md     全年 25 项任务全集
├── completed/              月度完成账本
├── env/                    环境、编译、实例管理
├── datatype/               数据类型与内部编码
├── architecture/           事件循环、对象系统
├── memory/                 内存管理与淘汰策略
├── persistence/            RDB / AOF 持久化
├── replication-ha/         主从复制与 Sentinel
├── cluster/                Cluster 集群
├── observability/          监控与排障
├── security/               ACL 与安全加固
├── engineering/            缓存架构、分布式锁
└── opensource/             Valkey 与云托管对比
```

## 任务体系

- 每个专题任务一个目录：`<TASK_ID>.idx.md`（状态）+ 文章 + `evidence/`（实验与源码取证）
- 权重与积分：S=1 / M=2 / L=3 / XL=5

## 进度

- 2026-08：ENV-001~003 / DAT-001~004 / OBS-001~004 / ENG-001 / PER-001~002 / ARC-001 已完成 16/25 项（详见 `study_record/completed/2026-08.md`）
