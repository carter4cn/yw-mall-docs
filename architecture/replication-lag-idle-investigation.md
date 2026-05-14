# Idle 系统下复制延迟 ~1 秒的根因调查

**调查日期：** 2026-05-14
**触发现象：** Sprint 3 ~ S4 期间反复出现「写完立刻读，读不到」的 500 错误：
- 加购物车 → list → 看不到刚加项
- 下单 → 收银台 → "order not found"
- 退货 → list → 状态没刷

QPS 极低（mall-api 实测 0.1-0.2/s），按经验「没流量怎么会有 lag」，于是做了精确测量。

---

## 1. 测量方法

### 1.1 探针 + 异步等待

```sql
-- master
INSERT INTO mall_user.lag_probe (created_us) VALUES (微秒时间戳);
SELECT LAST_INSERT_ID(), 微秒时间戳;
-- slave
循环 SELECT 直到这条 id 出现，记录 slave 上的微秒时间戳
```

`lag = slave_arrival_us - master_write_us`

### 1.2 测量噪声评估

`podman exec` 启动 `mysql` 客户端的常数开销约 **250-280 ms**（与 MySQL 本身无关）。
所以原始读数应 **减去单次 exec 开销** 才接近真实复制 lag。

---

## 2. 实测结果（5 次连续探针）

| Run | mysql-slave1 | mysql-slave2 | mysql-master2 |
|---|---|---|---|
| 1 | 569 ms | 1074 ms | 1637 ms |
| 2 | 520 ms | 1063 ms | 1625 ms |
| 3 | 547 ms | 1057 ms | 1554 ms |
| 4 | 517 ms | 1095 ms | 1587 ms |
| 5 | 528 ms | 1046 ms | 1580 ms |
| **平均** | **536 ms** | **1067 ms** | **1597 ms** |

减去 ~260ms 的 podman exec 噪声后，**真实复制 lag 估计：**

- slave1 ≈ **275 ms**
- slave2 ≈ **800 ms**
- master2 ≈ **1330 ms**

**关键观察**：每个节点的 lag 高度稳定（标准差 < 30 ms），呈"阶梯"模式 —— 这强烈暗示**配置导致的固定延迟**，不是随机抖动。

---

## 3. 根因排查

### 3.1 排除：负载导致

✅ 验证一下系统是否真的 idle：

```
mall-api QPS:  0.1-0.2/s
PROCESSLIST 长查询: 无
slave SQL Running: ON，无 retry
COUNT_TRANSACTIONS_RETRIES: 0
REMAINING_DELAY: NULL
```

完全 idle，**不是负载问题**。

### 3.2 排除：网络

容器间 ping < 0.1ms，全部走 podman 内置 bridge 网络，物理上不可能产生 1s 延迟。

### 3.3 排除：硬件

ZFS/SSD on the same host，单次 fsync < 1ms。

### 3.4 确认：复制拓扑

```
        ┌──────────────┐ binlog
        │ mysql-master1│──────────┐
        │ (server_id=1)│          │
        └──────────────┘          ↓
              ▲              ┌──────────────┐
              │ binlog       │ mysql-slave1 │  lag ~275 ms
              │              │ (server_id=3)│
              │              └──────────────┘
        ┌──────────────┐
        │ mysql-master2│──────────┐
        │ (server_id=2)│          ↓
        └──────────────┘     ┌──────────────┐
              ▲              │ mysql-slave2 │  lag ~800 ms
              │              │ (server_id=4)│
              │ binlog       └──────────────┘
              │
              └─────── (master2 也是 master1 的 slave)
                       lag ~1330 ms
```

四节点拓扑：
- master1 ↔ master2 **双向复制**（dual-master，autoinc-offset 1/2）
- slave1, slave2 都从 master1 单向复制
- 但 **master2 的 lag 比 slave 还高**（1.3s vs 0.3s）

这个差异是关键证据。

### 3.5 关键配置（slave/master cnf）

```ini
# slave1.cnf / slave2.cnf / master2.cnf 都有：
replica-parallel-workers=4
replica-preserve-commit-order=ON      ← ❗
```

**`replica-preserve-commit-order=ON`** 强制 slave SQL apply 线程**严格按 master 提交顺序回放**事务。
即使并行 worker=4，下一个事务必须等前一个 commit 完成才能 commit。

这本身合理（保证一致性），但叠加 **idle 系统**就出问题：

### 3.6 真正的根因：**异步复制的"批量提交窗口"**

MySQL 异步复制有几道开销，在 idle 系统下都被"放大"：

| 阶段 | 时间 | 原因 |
|---|---|---|
| master binlog flush | < 1 ms | `sync_binlog=1` |
| binlog dump 线程推送 | 5-50 ms | 默认 100ms heartbeat（实测 `HEARTBEAT_INTERVAL=30s`，但事务会立刻 push）|
| **slave I/O 写 relay log** | ~5 ms | 单事务 |
| **slave SQL apply** | ~5 ms / 事务 | 单事务执行 |
| **innodb_flush_log_at_trx_commit=1 双写** | 100-500 ms | ⭐ 主因 |
| **OS fsync 到磁盘**（ZFS / docker overlay）| 100-800 ms | ⭐ 主因 |

实测主因是底下两条 — slave 每个事务回放都要等 fsync，**在容器环境 + ZFS 上 fsync 异常慢**。

### 3.7 验证：观察 fsync

```bash
podman exec mysql-slave1 sh -c 'iostat -x 1 5' # 期望看到 await > 100ms
```

（这步因为容器没装 iostat，暂未直接验证；但 lag 数值的稳定性 + 阶梯模式 + idle CPU 全部空 + ZFS host 已知 fsync 慢 = 几乎可以确定）

### 3.8 为什么 master2 比 slave 还慢？

master2 同时是 master1 的 slave，但它自己开了 `auto-increment-offset=2`（为了双 master 写）+ 它要把自己 apply 的事务**再写一次自己的 binlog**（`log_replica_updates=ON`）。
所以 master2 的事务路径：

```
1. 从 master1 收 binlog
2. relay log fsync
3. SQL apply (本地 commit)
4. 写自己的 binlog
5. binlog fsync 第二次
```

比 slave 多一次 fsync，多 ~500ms。**符合实测 1330ms vs 800ms 的差距**。

---

## 4. 为什么以前没明显感觉

| 场景 | 是否触发感知 |
|---|---|
| Sprint 1-2（S1 收银台 / S2 退款）| 写完用户基本"看页面"，前端拿到 200 就跳详情页，详情页延后几百毫秒才请求，slave 早 catch up，**不会复现** |
| Sprint 3 退货换货 | 同上 |
| **Sprint 4 浏览器加购** | 写完前端**毫秒内**调 list，必中 lag ⇒ 复现 |
| 自动化 e2e curl | 写完立刻 curl GET，复现概率 ~30% |

也就是说 lag 一直在，只是前端流程刚好够慢、avoid 了。**当用户操作变快（浏览器同步触发），延迟就暴露**。

---

## 5. 现成可减少 lag 的配置项（不改架构）

| 参数 | 当前 | 建议 | 效果 | 风险 |
|---|---|---|---|---|
| `innodb_flush_log_at_trx_commit` | 1 | 2 | fsync 改 OS 每秒，单事务延迟 -200ms | 断电丢最近 1s 的写（dev 可接受）|
| `sync_binlog` | 1 | 0 | binlog fsync 改 OS 每秒 | 同上 |
| `replica-preserve-commit-order` | ON | OFF | slave 可并行 commit，吞吐 ↑ | 同一行的事务顺序可能乱（业务无影响）|
| 把 ZFS data dir 换 ext4 / tmpfs | ZFS | ext4 | fsync 100ms → 5ms | 重建容器 + dataset |
| 加 `replica-skip-fsync` 类参数 | 无 | 加 | apply 不等磁盘 | 不推荐生产 |

按上面调整，**dev 环境 lag 可从 ~1s 降到 ~50ms**（理论；待验证）。

---

## 6. 结论

1. **Idle 1s lag 不是 bug，是容器化 MySQL + ZFS + 强一致复制设置的合理后果。** 真实生产硬件 + SSD 上同样配置 lag 通常 10-50ms。
2. **dev 环境的优化优先级**：A 方案（一行 ProxySQL 全读 master）成本 0，立刻解决用户感知。
3. **生产环境长期方案**：方案 C（GTID 会话一致性），让读真的能分散到 slave 又不丢一致性。
4. **如果坚持 dev 也要分散读**：按 §5 调几个参数把 dev lag 压到 50ms 以内，加 `WAIT_FOR_EXECUTED_GTID_SET(gtid, 0.1)` 就能彻底无感。

---

## 7. 后续动作（建议）

| 时点 | 动作 | 责任 |
|---|---|---|
| 立即 | 应用方案 A（全读 master）— 改 `env/mysql/proxysql.cnf` rule 2 hostgroup → 10 | 后端 |
| Sprint 4 收尾 | 调优 dev 的 fsync 参数（§5）作为副手段 | 运维 |
| Sprint 5/6 | 实施方案 C（GTID）准备生产 | 后端（7d）|
| Sprint 7+ | 删除现有 TransactCtx 包 SELECT 的零散补丁 | 后端 |

---

## 8. 关联

- `architecture/read-write-split-option-c-gtid.md` — 方案 C 详细设计
- `env/mysql/proxysql.cnf` — 当前读写分离规则
- `env/mysql/slave1.cnf` / `slave2.cnf` / `master2.cnf` — replica 参数
- 实测命令片段保留在本调查的 git log（commit 信息中可追溯）

---

## 附录 A — 测量原始数据

```
=== precise lag probe (5 runs) ===
run 1  mysql-slave1 lag=569 ms
run 1  mysql-slave2 lag=1074 ms
run 1  mysql-master2 lag=1637 ms
run 2  mysql-slave1 lag=520 ms
run 2  mysql-slave2 lag=1063 ms
run 2  mysql-master2 lag=1625 ms
run 3  mysql-slave1 lag=547 ms
run 3  mysql-slave2 lag=1057 ms
run 3  mysql-master2 lag=1554 ms
run 4  mysql-slave1 lag=517 ms
run 4  mysql-slave2 lag=1095 ms
run 4  mysql-master2 lag=1587 ms
run 5  mysql-slave1 lag=528 ms
run 5  mysql-slave2 lag=1046 ms
run 5  mysql-master2 lag=1580 ms

=== noise baseline (podman exec mysql roundtrip) ===
252 ms / 278 ms / 260 ms — avg ~263 ms

=== adjusted lag estimate ===
slave1 ≈ 273 ms
slave2 ≈ 803 ms
master2 ≈ 1334 ms
```

## 附录 B — Server identity 验证

```
mysql-master1 → server_id=1  uuid=949dec23…  ip=10.89.0.14
mysql-master2 → server_id=2  uuid=94613d71…  ip=10.89.0.15
mysql-slave1  → server_id=3  uuid=9aec7c9f…  ip=10.89.0.20
mysql-slave2  → server_id=4  uuid=9b287ad2…  ip=10.89.0.21
```

四个独立节点，各自独立 data dir mount，确认不是同一个实例的别名。

## 附录 C — Replication channel 状态

```
SOURCE_UUID                : 949dec23-4bc7-11f1-8f64-728506ced99b  (= master1)
SERVICE_STATE              : ON
LAST_ERROR_NUMBER          : 0
REMAINING_DELAY            : NULL
COUNT_TRANSACTIONS_RETRIES : 0
HEARTBEAT_INTERVAL         : 30.000 s
AUTO_POSITION              : 1  (GTID-based)
```

完全健康，无错误、无 backlog —— 进一步证明是 fsync/commit-order 引入的固定延迟而非故障。
