# 数据库复制 / 路由

特点：写完立刻读读不到 / 读老数据 / "no rows in result set" 偶发。
所有这一类都源于同一根因（ProxySQL 把 SELECT 路由到延迟的 slave），但**触发现象散落在不同 RPC 里**。

---

## #1 cart-rpc ListItems read-after-write — 加完购物车看不到

**症状**：浏览器加购物车后立刻 list，60% 概率拿不到刚加的商品；几次后又突然全出来。
**触发**：高频，浏览器同步发请求时必中。
**根因**：ProxySQL 默认把 SELECT 分发到 hg=20（master + 2 slave），slave 复制延迟 ~500ms-1.3s，导致 read-after-write 不一致。
**修复**：
1. **第一版**（治标）— commit `4398890`：cart-rpc ListItems 用 `TransactCtx` 包 SELECT，事务自动 pin master。
2. **架构级修复**（治本）— `env` repo commit `3a47a38`：ProxySQL rule_id=2 destination_hostgroup 20 → 10，所有 SELECT 走 master。
**如何避免**：方案 A 实施后，新 RPC 默认走 master 不会再撞；等真上方案 C（GTID 会话一致性，见下文）后才能安全地把读分散。
**相关**：
- `architecture/replication-lag-idle-investigation.md`（lag 实测 + 根因）
- `architecture/read-write-split-option-c-gtid.md`（GTID 方案设计）

---

## #2 payment-rpc GetCashier 偶发 "order not found"

**症状**：下单后立刻进收银台 → 500 "order not found"；2 秒后手动重试 → 200。
**触发**：从 createOrder 直接 redirect 到 cashier 时必现；用户慢一点点击就不一定。
**根因**：同 #1 — payment-rpc 读 order 表也是 plain SELECT 走 slave。
**修复**：commit `844034f` — TransactCtx 包 SELECT。后续方案 A 实施后这个补丁严格说也已多余。
**如何避免**：同 #1。

---

## #3 cart-rpc 写后 list 间断"穿越式"返回旧 + 新数据

**症状**：连续 add A / add B / list 时，list 可能：
- 只看到 A
- 看到 A + B
- 看到 A + B + C（C 是上次 cart 的残留）
**触发**：4 个 SELECT/写交叉时偶发，不可重现 100%。
**根因**：ProxySQL 在 slave 之间负载均衡，每个 slave lag 不同（slave1 ~500ms / slave2 ~1100ms / master2 ~1600ms），同一 session 不同请求落在不同 slave 看到的数据快照不同 → "穿越"感觉。
**修复**：方案 A 一次性消掉。
**如何避免**：永远不要假设同一个 session 内连续读看到的是同一份数据 —— 异步复制下完全可能"前后矛盾"。

---

## #4 idle 系统也有 1-2s 复制延迟（不是负载问题）

**症状**：完全 idle 的 dev 系统，slave 也稳定延后 master 1 秒左右。
**触发**：Sprint 4 浏览器购物链路打通后才暴露 —— 写完用户立刻读必中。
**根因**（详见 `architecture/replication-lag-idle-investigation.md`）：
- `replica-preserve-commit-order=ON` 强制串行 commit
- `sync_binlog=1` + `innodb_flush_log_at_trx_commit=1` 每 commit 双 fsync
- 容器 ZFS / overlayfs 的 fsync 在没负载时也要 100-500ms
- master2 还多一次 binlog fsync（既 apply 又写自己 binlog）
**修复**：
1. **现在**（治标）— 方案 A，全 SELECT 走 master。
2. **未来**（治本）— Sprint 5/6 上方案 C（GTID 会话一致性，7 人天预算）。
**如何避免**：
1. 永远不要相信"我系统没流量所以肯定无延迟"。生产硬件 SSD 通常 lag 10-50ms，但容器 + ZFS dev 环境放大到 1s。
2. 写代码不依赖任何"slave 应该已经看到最新数据"的假设。
3. 如非要保留读写分离，加 fsync 调优（详见调查文档 §5）能把 lag 压到 ~50ms。
**相关**：
- `architecture/replication-lag-idle-investigation.md`
- `architecture/read-write-split-option-c-gtid.md`
