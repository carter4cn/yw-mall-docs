# Placeholder 占位代码混进了关键路径

特点：早期 stub 用 wd-empty / `count++` / nil-return 等占位，时间一长被当成"做完了"，
真实流程触发后才发现根本没接通。

这类 bug 和 schema drift 类似 —— 不是写错，而是**漏写**；很难通过 review 发现，必须靠完整 e2e。

---

## #1 cart 加购物车只改本地 ref，不调后端

**症状**：用户点"加入购物车" → toast 提示"已加入" → 进购物车 tab 看到的永远是空的。
**触发**：必现。
**根因**：`pages/product/detail.vue` 的 addToCart 只调用 `cartStore.increment()`，store 里 `count` 是个 ref，没有 items 列表也没调过 `/api/cart/*`。`src/api/` 目录甚至没 `cart.ts` 文件。
**修复**：commit `25f527c` — 新建 api/cart.ts；store 改为 CartItem[] + computed count；detail.vue addToCart 改 `await cartStore.add()`。
**如何避免**：
1. 占位代码要清楚标注 `// TODO: backend wire-up` 或在文件里 `wd-status-tip status="placeholder"`，让 grep 一搜就出。
2. CLAUDE.md 已经写明"cart.ts is count only"，但没人盯着 → 应改成 README 顶部的"未完成功能"清单+死线。
3. 每个 sprint 收尾前过一遍"占位文件清单"。

---

## #2 cart/index.vue 是 wd-status-tip "购物车功能即将上线"

**症状**：跟 #1 同链路，购物车 tab 永远显示"购物车功能即将上线"。
**触发**：必现。
**根因**：placeholder 一个组件就交差。
**修复**：commit `25f527c` 一并补 list / 数量编辑 / 删除 / 合计 / 全选 / 去结算。
**如何避免**：见 #1。

---

## #3 address/list.vue 是 "功能即将上线" — 阻断结算流程

**症状**：去结算 → mall-api CreateOrder 报 "收货地址不存在"，但「我的地址」页面是占位 "功能即将上线"，用户没法添加地址 → 流程死锁。
**触发**：新用户（没有 SQL 手工塞默认地址）必现。
**根因**：address pages 之前是 wd-status-tip 占位；后端路由 + types + logic 都齐了，只差前端没接。
**修复**：commit `2f6ec16` — list.vue 真接通 + 新建 edit.vue + api/address.ts + 路由注册。
**如何避免**：
1. 占位页若**阻塞主链路**，必须排到当前 sprint —— 不能用"未来上线"借口拖。
2. 整个流程跑一遍 e2e（注册新用户 → 加购 → 下单 → 支付 → 看订单），任何一步卡住都是 P0。
