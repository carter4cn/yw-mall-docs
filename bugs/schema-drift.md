# Schema Drift / 列与代码错位

特点：DDL 加了列 / 改了类型，但读路径的 SELECT / scan struct / parser 没跟上 →
500 "not matching destination" / "no rows" / 解析后字段全错。

---

## #1 ListOrders 500 "not matching destination to scan"

**症状**：`/api/order/list` 接口稳定返回 500，error 是 sqlx "not matching destination to scan"。
**触发**：必现，所有用户访问"我的订单"都炸。
**根因**：S1.5 给 order 表加了 5 个时间线列（pay_time / ship_time / complete_time / cancel_time / cancel_reason），同文件 `orderRow` struct 已加这些字段（共 18 个 db tag），但 `ListOrders` 的 SELECT 还在选老的 13 列；sqlx 拿 13 个返回值映射 18 字段 struct → 报 "not matching"。
**修复**：commit `844034f` — 复用同文件 `orderTimelineCols` 常量；resp 透传 5 个时间线字段。
**如何避免**：
1. 加列时，**全文搜** `SELECT ... FROM \`order\``，把所有 SELECT 一并改完再提交。
2. 把 SELECT 列单提取成 `const xxxCols = "..."` 复用，已经是良好习惯（getorderlogic.go 就是这样），只是 ListOrders 没沿用。
3. 长期：用 sqlx `*` SELECT + struct field 自动映射；或者改 sqlc 生成 query。

---

## #2 splitPerms 把 JSON 字符串 ["*"] 当 CSV

**症状**：admin login 返回 `permissions:["[\"*\"]"]`。
**触发**：必现。
**根因**：DB `admin_user.permissions` 列存 JSON 字符串，应用层 splitPerms 用 `strings.Split(",")` 当 CSV 切。
**修复**：commit `6430f94`。
**如何避免**：见 frontend-backend-contract.md #4。

---

## #3 SubmitRefundRequest INSERT 漏写 refund_type 列

**症状**：S3 期间所有 refund_type=2/3 的请求 DB 里都存成 1。
**触发**：必现，refund_type != 1 时。
**根因**：proto 加了字段、DB 加了列、struct 加了 field，但 INSERT SQL 列清单没改。
**修复**：S3 daily 里的修复 — INSERT 加上 `refund_type` 列 + 用 `refundTypeOrDefault()` helper 兜底 0 → 1。
**如何避免**：
1. 加列后，**搜全部** `INSERT INTO {table}` / `UPDATE {table}` / `SELECT * FROM {table}` 是否都跟上。
2. 长期：用 sqlc / xorm 等 ORM 让"加字段"成为单点改动。
**相关**：daily/2026-05-14.md S3 章节
