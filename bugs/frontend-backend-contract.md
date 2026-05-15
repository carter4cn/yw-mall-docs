# 前后端契约不对齐

特点：HTTP 200 但 UI 渲染异常、字段拿到 undefined 兜底成空、跳转死循环。
没有任何后端报错可以排查 —— 必须翻代码或抓包对比。

---

## #1 cart 登录死循环 — store 字段名 token vs accessToken

**症状**：登录成功（接口返回正确 token），页面看似没跳转，实际是被 cart tab 弹回 login 形成死循环。
**触发**：必现，登录后访问任何 tab 页都会失败。
**根因**：cart/index.vue 检查 `userStore.accessToken`，但 `stores/user.ts` 暴露的字段叫 `token`（`accessToken` 是 setSession 入参，未挂到 store 上）。`accessToken` 始终 undefined → 跳 login → switchTab 回 cart → 死循环。
**修复**：commit `91c95ef` — `accessToken` → `token`。
**如何避免**：
1. store 模块明确暴露列表，TS 类型严格化；引用前 IDE 应能报红。
2. 任何"页面看起来没动作"先打开 DevTools Network/Console，看是不是页面在反复跳。
3. 全文搜 `userStore\.` 看所有调用是否字段名一致。
**相关**：登录 P0 (yw-mall-docs/feat/login-revamp.md)

---

## #2 address 列表添加成功但 UI 空 — 接 resp.items / 后端是 addresses

**症状**：`/api/address/add` 200，但 `/api/address/list` 之后地址列表 UI 一直显示"空"。
**触发**：必现。
**根因**：前端 `api/address.ts` 类型 `request<{ items: AddressItem[] }>`，list.vue 取 `resp.items`；后端 `ListAddressesResp.Addresses` JSON 字段名 `addresses`。`resp.items` 永远是 undefined，`?? []` 兜底成空数组。
**修复**：commit `d2512b0` — `items` → `addresses`。
**如何避免**：
1. 前端类型 interface 直接复制后端 `types.go` 字段名（不要拍脑袋）。
2. 新接口接通后**至少抓一次实际响应** JSON 对照前端类型。
3. 长期：从 mall.api DSL 自动生成 TS 类型（消除人写双份的错位）。

---

## #3 mall-api CreateOrder 透传 AddressId=0 — 后端报"收货地址不存在"

**症状**：浏览器去结算 → 500 `收货地址不存在`。
**触发**：用户没有任何地址，或前端没传 addressId 时必现。
**根因**：mall-api `CreateOrder` 之前只透传 `Items`，`AddressId` 字段保持默认 0；order-rpc 拿到 `AddressId=0` 调 `user-rpc.GetAddress(uid, id=0)` 必然 not found。
**修复**：commit `7c39cf7` — mall-api 在调 OrderRpc 前先 `GetDefaultAddress(uid)`，拿到 `addr.Id` 再传。空地址给中文错误"请先添加默认收货地址"。
**如何避免**：
1. proto 里强一致字段（如 AddressId）应在 mall-api 层校验非空再透传。
2. e2e 至少跑过一遍"零地址用户下单"场景。
3. 长期：让 order-rpc 自己读默认地址（业务收口），mall-api 只传 uid。

---

## #4 admin-fe permissions 显示 `["[\"*\"]"]`

**症状**：admin login 返回的 `permissions` 字段是个嵌套字符串。
**触发**：必现，所有 admin 账号。
**根因**：DB 列 `admin_user.permissions` 存 JSON `'["*"]'`，admin-api 的 `splitPerms` 直接 `strings.Split(",")` 当 CSV 切，把整个 JSON 字符串当成一个 perm。
**修复**：commit `6430f94` — 先检查 `[` 前缀 → `json.Unmarshal`，失败 fallback CSV。
**如何避免**：
1. 列存什么类型 / 应用层用什么解析器，应在表 DDL 注释或 ADR 里声明。
2. 任何来自 DB 的 string 字段，加日志检查实际内容再决定怎么 parse。
3. 长期：admin_user.permissions 改成 `JSON` 列类型，driver 层就报错而不是默默接受字符串。

---

## #5 mall-api SubmitRefundReq 缺 RefundType 字段

**症状**：S3 时 C 端 refund 接口透传 `refundType` 始终丢失，DB 里永远是默认值 1。
**触发**：refundType != 1 时必现。
**根因**：executor 改了 proto 加 `refundType` 字段、加了 DB 列、改了 INSERT，但 mall-api 的 `types.SubmitRefundReq` struct 没加这字段，HTTP 层根本接不到。
**修复**：S3 daily 提到的 commit；types + logic 双向加 `RefundType int32 \`json:"refundType,optional"\``。
**如何避免**：
1. 加 proto 字段时必须连 mall-api 的 types.go 一起改（同一 PR / commit）。
2. 长期：从 .proto 自动生成 mall-api types（goctl 已支持，开起来）。
**相关**：daily/2026-05-14.md S3 章节
