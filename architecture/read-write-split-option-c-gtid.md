# 方案 C — GTID 会话一致性（生产级读写分离正解）

**编写日期：** 2026-05-14
**状态：** 草案 / 待评审
**关联：**
- `architecture/replication-lag-idle-investigation.md`（idle 系统下 ~1s lag 的根因）
- `feat/login-revamp.md`（不直接相关但同一基础设施栈）

---

## 1. 问题陈述

当前 yw-mall 用 ProxySQL 做 MySQL 读写分离：

```
mysql_query_rules:
  rule 1: SELECT … FOR UPDATE   → hostgroup 10 (master)
  rule 2: SELECT *               → hostgroup 20 (master + slaves)
  default                        → hostgroup 10 (master)
```

异步复制 + 读分发到 slave 必然存在 **read-after-write 不一致窗口**（实测 idle ≈ 0.5-1.6s，见调查文档）。
典型场景：

- 加购物车 → 立刻 list ⇒ 看不到刚加的商品
- 下单 → 立刻进收银台 ⇒ "order not found"
- 退款 → 立刻 list ⇒ 状态没刷
- 改密码 → 立刻登录 ⇒ 旧密码还能登

打补丁（每个读路径手工 `TransactCtx` 包 SELECT 强制 pin master）治标不治本：

- 每加一个 read 路径就要记得包
- 老代码忘了包，bug 会以"偶发 500/no-rows"形式蔓延
- 包成事务后 master 还是扛了全部读，分散读副本的初衷丢了

**方案 C 是业界生产环境（淘宝 / 蚂蚁 / 阿里 RDS / Vitess / TiDB Async Read 等）真正在用的方法。**

---

## 2. 技术原理

### 2.1 GTID 简述

**GTID (Global Transaction Identifier)** 是 MySQL 5.6+ 给每个写事务分配的全局唯一 ID：

```
949dec23-4bc7-11f1-8f64-728506ced99b:14183
└────────────── source_uuid ──────────────┘ └ txn # ┘
```

- 每个事务都有一个 GTID，主从复制时 GTID 跟着事务流到所有副本
- slave 上 `gtid_executed` 变量记录已 apply 的 GTID 集合（`GTID_SUBSET`）
- 应用层拿到 master 写完返回的 GTID，再发到任意 slave 查 `WAIT_FOR_EXECUTED_GTID_SET(gtid, timeout)`，slave 等到 catch up 后才返回

### 2.2 方案 C 端到端流程

```
─── 写路径 ────────────────────────────────────────────
Client → mall-api → user-rpc UpdatePassword
                           ↓
                   sqlx.TransactCtx (master)
                     INSERT / UPDATE …
                     SELECT @@GLOBAL.gtid_executed → "uuid:1-X"
                   ← 返回 last_gtid = "uuid:X"
                           ↓
                   middleware 把 last_gtid 写到 ctx
                           ↓
                   mall-api 把 last_gtid 写到 user session
                     redis: session:{token}.lastGtid = "uuid:X"
                   ← HTTP 200

─── 读路径 ────────────────────────────────────────────
Client → mall-api → SessionAuthMiddleware
                     从 session 读 lastGtid
                     注入到 ctx
                           ↓
                   → user-rpc.GetUser
                           ↓
                   sqlx 中间层：
                     SELECT WAIT_FOR_EXECUTED_GTID_SET(@lastGtid, 0.5)
                                                ┌── 0 = 已 catch up ✓
                     ↓                          └── 1 = 超时 → fallback
                     case 0: SELECT * FROM user  (任意 slave)
                     case 1: 改路由到 master 再 SELECT
                           ↓
                   ← 返回数据 + 新的 gtid（如果是写事务）
```

### 2.3 关键组件

| 组件 | 职责 |
|---|---|
| **mall-common/dbx** 中间件 | 封装 sqlx：写后捕获 gtid；读前注入 wait |
| **gtid context-key** | go-zero ctx 里塞 lastGtid，跨进程靠 gRPC metadata 透传 |
| **mall-api SessionMiddleware** | 把 session.lastGtid 灌进 ctx，写完再回写 session |
| **gRPC interceptor** | 自动在 outgoing metadata 加 `x-gtid: …`；服务端解析回 ctx |
| **ProxySQL hint** | 用 `/* min_gtid_set=… */` 注释让 ProxySQL 自动路由（高级选项，可选）|

---

## 3. 修改成本（人天估算）

### 阶段 P0：基础设施（3 天）

| Story | 人天 | 内容 |
|---|---|---|
| C0.1 mall-common 加 `gtid` 包 | 0.5d | ctx key 定义、Get/Set helper、序列化 |
| C0.2 sqlx 拦截器：写后捕获 gtid | 1d | wrap TransactCtx + ExecCtx，写完 `SELECT @@gtid_executed` |
| C0.3 sqlx 拦截器：读前等待 gtid | 1d | wrap QueryCtx：先 `WAIT_FOR_EXECUTED_GTID_SET(gtid, 0.5)`，超时 fallback 改走 master |
| C0.4 单测 + 故障注入测试 | 0.5d | mock GTID 测试 wait 超时回退、空 gtid 跳过 wait 等边界 |

### 阶段 P1：gRPC 透传（2 天）

| Story | 人天 | 内容 |
|---|---|---|
| C1.1 gRPC client interceptor | 0.5d | outgoing metadata 加 `x-gtid` |
| C1.2 gRPC server interceptor | 0.5d | 解析 `x-gtid` → ctx |
| C1.3 mall-api 集成 | 0.5d | login/refresh 把 session.lastGtid 同步 ctx + 写完回写 session |
| C1.4 e2e 验证 | 0.5d | 多 RPC 链路 (mall-api → cart-rpc → user-rpc) 跨服务 gtid 透传 |

### 阶段 P2：上线灰度 + 监控（2 天）

| Story | 人天 | 内容 |
|---|---|---|
| C2.1 metrics：gtid wait p99 / fallback 比例 | 0.5d | Prometheus counter + histogram |
| C2.2 灰度开关 | 0.5d | feature flag：`USE_GTID_CONSISTENCY=true/false` |
| C2.3 Grafana 看板 | 0.5d | 「读 wait 耗时」「fallback 比例」「lag p50/p99」 |
| C2.4 文档 + Runbook | 0.5d | 现网调参、回退步骤、常见问题 |

### 阶段 P3（可选）：删除老补丁 + 重构

| Story | 人天 | 内容 |
|---|---|---|
| C3.1 删除现有 TransactCtx 包 SELECT 的"读补丁" | 1d | cart-rpc, order-rpc 已知；扫一遍其他 RPC |
| C3.2 训练同事新模式 | 0.5d | 团队 share + cookbook |

**累计：P0+P1+P2 = 7 人天 ≈ 1.5 周（1 后端）**
**含 P3 重构：8.5 人天 ≈ 2 周**

---

## 4. 收益分析

### 4.1 一致性

| 维度 | 当前（部分包 master） | 方案 C |
|---|---|---|
| read-after-write | 看代码记没记得包；包不全的地方踩坑 | 保证（GTID wait + master fallback）|
| 跨服务一致性（A 写 → B 读 ）| 没有任何保证 | 保证（gRPC 透传 gtid）|
| 副本失联场景 | 读 fallback master，无差别可用 | 同上，可用 |

### 4.2 性能

| 指标 | 当前 | 方案 C |
|---|---|---|
| 读 RT（slave 已 catch up） | 1ms | 1ms + GTID wait 检查 ≈ 1.0-1.2ms |
| 读 RT（slave 慢） | 1ms（脏读） | wait 到 timeout 然后 fallback master，最多 +500ms |
| 写 RT | 1ms | 1ms + 捕获 gtid (~0.2ms) ≈ 1.2ms |
| master CPU/QPS 占比 | 50% 左右 | 35% 左右（读真的能分散）|

### 4.3 运维

| 项 | 当前 | 方案 C |
|---|---|---|
| 新 RPC 写代码 | 必须想"会不会撞 lag"决定要不要包事务 | 不需要想，sqlx wrapper 全自动 |
| 调试 | "为什么我刚写完就读不到？" 难复现 | wait 超时 fallback 路径有 metric，可观测 |
| 故障定位 | slave lag 高时大面积报错 | wait 超时回退，UX 不受影响 |

---

## 5. 实现关键点

### 5.1 哪些 SELECT 必须 wait gtid？

**必须**（用户语义一致性）：
- 同一用户在同一 session 内的所有读

**不必须**（弱一致 OK）：
- 后台报表查询
- 商品列表、店铺列表等公共读（用户读到 1-2s 前的视图无感）

实现上把 wait 做成默认开启 + per-call 用 ctx 跳过：

```go
// 默认开启
db.Get(ctx, &user, "SELECT … WHERE id=?", uid)

// 显式跳过（报表、低优先级读）
ctx = gtid.SkipWait(ctx)
db.Get(ctx, &row, "SELECT …")
```

### 5.2 GTID 存哪？

| 存储 | 优点 | 缺点 |
|---|---|---|
| **HTTP cookie** | 0 服务端状态 | 必须在客户端，跨服务靠 metadata 透传 |
| **Redis session**（推荐）| 配合 P0 session 已有；服务端控制 | 写 + 读 都要碰 Redis（已有路径） |
| **gRPC ctx metadata only** | 简单 | 一旦客户端不带，回退到无 gtid |

**推荐**：Redis session 持久化 + ctx metadata 传递（双层保险）。

### 5.3 WAIT_FOR_EXECUTED_GTID_SET 的成本

实测每次 wait 0-50ms（slave 已 catch up 时立即返回）。在 yw-mall idle 实测 lag ~1s 的极端情形下，需要 200-500ms timeout。

```sql
-- 0 = 等到了，1 = 超时
SELECT WAIT_FOR_EXECUTED_GTID_SET('uuid:1-14183', 0.5)
```

### 5.4 fallback 策略

```go
result := slaveDB.QueryWithGtidWait(ctx, gtid, timeout=500ms)
if result.timedOut() {
    metrics.GtidFallback.Inc()
    return masterDB.Query(ctx)
}
return result
```

---

## 6. 风险与缓解

| 风险 | 影响 | 缓解 |
|---|---|---|
| GTID 透传中断（如某 RPC 没接 interceptor）| 该读路径回退到无一致性保证 | metrics 监控 "无 gtid 的读" 比例；CI 检查所有 RPC 都注册 interceptor |
| slave 长时间 lag → 大量 fallback master | master 突发负载 | metrics 告警 fallback > 10%；自动 slave 退队 + alerts |
| 应用代码 ctx 没传到 sqlx | wait 没生效，等价于无方案 | go-zero linter 规则 / e2e 测试覆盖 |
| MySQL 8.0 之前 GTID 模式有 bug | 我们用 9.6 不受影响 | — |
| ProxySQL 配置漂移导致 hint 失效 | 静默回退到旧路由 | 启动时校验 ProxySQL config |

---

## 7. 替代方案对比

| 方案 | 强一致性 | 读分散 | 实现成本 | 适用阶段 |
|---|---|---|---|---|
| **A. 全读 master** | ✅ | ❌ | 5 min | MVP（QPS < 100）|
| **B. 半同步 + ProxySQL max_replication_lag** | ⚠ 亚秒级仍可能 | ✅ | 1 天 | 中型（QPS 100-1k）|
| **D. 按表静态路由** | ⚠ 维护负担 | ✅ | 半天 | 短期过渡 |
| **C. GTID session consistency** ⭐ | ✅ | ✅ | **7 天** | **生产（QPS 1k+，对外用户）** |

---

## 8. 推荐时间表

1. **立即（今天）**：执行方案 A（一行 ProxySQL 改动），把当前所有 read-after-write bug 消掉
2. **Sprint 5 / 6**：实施方案 C 的 P0+P1+P2（7 人天）
3. **Sprint 7+**：删除老的 TransactCtx 补丁（P3）+ 把读路由按表分级

这样可以避免现在投入 7 人天去搞 GTID 但 QPS 几百以内根本看不到收益的尴尬，
同时给团队留出真正灰度上线前的窗口实施方案 C。

---

## 9. 关联文档

- `architecture/replication-lag-idle-investigation.md` — 当前 idle lag 1s 的根因调查
- `planning/mvp_sprint_plan.md` — Sprint 排期参考
- ProxySQL 官方 GTID docs: https://proxysql.com/blog/proxysql-gtid-causal-reads/
- MySQL `WAIT_FOR_EXECUTED_GTID_SET`: https://dev.mysql.com/doc/refman/9.6/en/gtid-functions.html
