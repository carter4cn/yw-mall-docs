# yw-mall-docs

yw-mall 电商平台**唯一**的文档汇总仓库 — 产品需求、技术规划、工作日志、架构决策、运维手册、复盘记录都汇集在这里。

> 业务代码仓库（yw-mall / yw-mall-admin / yw-mall-admin-fe / yw-mall-fe / yw-mall-deploy）原则上不再单独维护 `docs/` 目录，避免双写同步问题。

---

## 目录结构

```
yw-mall-docs/
├── prd/                  # 产品需求文档
├── planning/             # 路线图、Sprint 计划、差距分析
├── feat/                 # 跨 sprint 特性方案 / 架构改造专题（含时间预算 + 跟踪表）
├── daily/                # 每日工作日志（按日期归档，跨 repo 合并）
├── architecture/         # 架构决策记录 (ADR) / 系统图 / 数据流图
├── api/                  # 统一 API 规范（OpenAPI / proto 参考）
├── runbooks/             # 运维手册 / 故障预案 / 上下线流程
└── retrospectives/       # 项目复盘 / Sprint 回顾
```

---

## 关键文档索引

### 产品需求

- [`prd/admin-merchant-portal.md`](prd/admin-merchant-portal.md) — 后台管理 PRD（管理员 + 店家门户）

### 规划与差距分析

- [`planning/platform_completeness.md`](planning/platform_completeness.md) — 整个电商平台完整度评估（C 端 + B 端 + 平台 + 基础设施）
- [`planning/commercial_readiness_gap.md`](planning/commercial_readiness_gap.md) — 后台专项商用差距分析（P3/P4/P5 路线图）
- [`planning/mvp_sprint_plan.md`](planning/mvp_sprint_plan.md) — MVP 12-Sprint 详细计划（6 个月、8 人团队）
- [`planning/q1q2_tracking.md`](planning/q1q2_tracking.md) — Sprint Story 跟踪表 + 角色甘特图

### 特性方案

- [`feat/login-revamp.md`](feat/login-revamp.md) — 登录体系改造（Opaque Token + Redis Session，参考淘宝/京东）— P0 / P1 / P2 / P3 时间预算 + 跟踪表

### 架构 / ADR

- [`architecture/read-write-split-option-c-gtid.md`](architecture/read-write-split-option-c-gtid.md) — 读写分离方案 C：GTID 会话一致性（生产级正解）— 原理 / 7 人天预算 / 收益对比 A/B/C/D
- [`architecture/replication-lag-idle-investigation.md`](architecture/replication-lag-idle-investigation.md) — Idle 系统下 ~1s 复制延迟根因调查（实测数据 + 拓扑分析 + 修复路径）

### 工作日志

- [`daily/2026-05-10.md`](daily/2026-05-10.md) — etcd 配置中心修复 + rebuild.sh
- [`daily/2026-05-12.md`](daily/2026-05-12.md) — P2 后台管理 9 个 Story 全部完成 + admin-fe 14 模块

---

## 写作约定

### 通用

- 一律使用 **Markdown**（GitHub Flavored Markdown）
- 文件命名：英文小写 + 连字符（`commercial-readiness-gap.md` 这种），**不要**空格/驼峰/中文
- 日期格式：ISO 8601 `YYYY-MM-DD`（`2026-05-12`）
- 表格 / 代码块 / 链接 比纯文本更受欢迎
- 头部 metadata 写明：版本、日期、适用范围、关联文档

### 工作日志（daily/）

每个工作日**一个文件**（`YYYY-MM-DD.md`），跨 repo 的工作合并：

```markdown
# 工作日志 · YYYY-MM-DD

> 涉及 repo：repo-a · repo-b · ...

## 📦 repo-a
（该 repo 当日内容）

## 🚢 repo-b
（该 repo 当日内容）
```

主题区分用 emoji + repo 名作为二级标题。这样的好处是：日历跳转 → 单页看到当天所有 repo 的工作。

### 规划文档（planning/）

包含**估算工时**或**优先级矩阵**时，请保持单位一致：
- 工时单位：**人天**（≈ 1 人 1 个工作日）
- 月度估算：**人月** = 22 人天
- 优先级：P0（必须立即做）/ P1（本季度）/ P2（下季度）/ P3-P5（参见 `commercial_readiness_gap.md` §2-4）

### 架构决策（architecture/）

按 [ADR](https://adr.github.io/) 格式：

```markdown
# ADR-NNNN: 标题

- 日期：YYYY-MM-DD
- 状态：proposed / accepted / superseded
- 决策者：@xxx

## 背景

## 决策

## 后果（正面 + 负面）

## 备选方案

## 关联
```

---

## 关联仓库

| Repo | 内容 |
|---|---|
| [yw-mall](https://github.com/carter4cn/yw-mall) | 13 个 mall-*-rpc 后端服务 + mall-api C 端网关 + mall-activity-async-worker |
| [yw-mall-admin](https://github.com/carter4cn/yw-mall-admin) | 后台管理 HTTP API 网关（admin + merchant 双路由） |
| [yw-mall-admin-fe](https://github.com/carter4cn/yw-mall-admin-fe) | 后台前端（admin/ + merchant/ 两个 SPA） |
| yw-mall-fe | C 端商城前端（uni-app H5） |
| [yw-mall-deploy](https://github.com/carter4cn/yw-mall-deploy) | podman-compose 编排 + 启动脚本 + DDL bootstrap |

---

## 贡献

直接 `main` 分支提交（小团队 / 个人项目模式）。如未来引入 review 流程：

1. 新建 `docs/feat-xxx` 分支
2. 完成文档 → PR
3. 自动 trigger markdown lint（暂未配，未来加 GitHub Actions）

不可逆变更（删 / 大规模重命名）走 PR + review。
