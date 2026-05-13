# 登录体系改造方案 — Opaque Token + Redis Session

**编写日期：** 2026-05-14
**状态：** **P0 已完成 2026-05-14** — 真实账号 alice/alice123 浏览器登录通；e2e (login → access → refresh → logout → 401) 全部 ✅
**关联文档：** `planning/mvp_sprint_plan.md`、`planning/q1q2_tracking.md`
**触发动因：** S1-S3 期间发现 user-rpc 签发的 JWT 在 mall-api 验签失败，疑似 `onConfigChange` 中 `yaml.Unmarshal` 静默丢字段（与 daily 05-10 修过的 configcenter loader bug 同源），所有 e2e 必须手工 openssl 签 token 绕开。
**决策：** 不修 JWT，整体替换为参考淘宝/京东的 **Opaque Token + Redis Session** 方案。

---

## 0. 当前痛点

| 痛点 | 影响 |
|---|---|
| JWT 一旦签发无法撤销 | 改密后无法强制下线；token 泄露窗口期长 |
| payload 可 base64 解码看到 uid/role | 信息泄露 |
| user-rpc hot-reload bug 把 AccessSecret 改成空 | 当前**根本登录不上** |
| 无法做多设备管理 | 无法实现「我的设备」/ 一键下线 |
| 无法实时风控介入 | 一旦下发不可控 |

---

## 1. 核心架构

### 1.1 不用 JWT，改用 Opaque Token + Redis Session

token 就是一段**随机字符串**（不带任何 payload），所有元数据存 Redis：

```
access_token  = base64url(crypto.RandomBytes(32))   # 128 bits 熵
refresh_token = base64url(crypto.RandomBytes(32))
```

```redis
SET session:{access_token} {
  uid: 1,
  username: "alice",
  role: "user",
  deviceId: "d_iphone_15_pro_xxx",
  ip: "1.2.3.4",
  csrfToken: "abc...",
  perms: ["product.read", "order.write"],
  loginTime: 1778683000,
  lastActive: 1778683500
}                                                   # TTL 30 min

SET refresh:{refresh_token} {
  uid, deviceId, rotateCount
}                                                   # TTL 7 天

SADD user_sessions:1 access_token_1 access_token_2 ...
```

### 1.2 端到端流程

```
Client → POST /api/auth/login {credentials}
       ← 200 {accessToken, refreshToken, expiresIn:1800, csrfToken}

Client → GET /api/order/list
         Authorization: Bearer <accessToken>
       ↓
mall-api SessionMiddleware:
  1. token = header.Bearer
  2. session = redis.GET session:{token}
  3. if !session: 401
  4. redis.EXPIRE session:{token} 1800   # 滑动续期
  5. ctx.uid = session.uid
       ↓
handler 用 ctx.uid

Client → POST /api/auth/refresh {refreshToken}
       ← 200 {newAccessToken, newRefreshToken}    # 旋转策略

Client → POST /api/auth/logout
       → DEL session:{token} + DEL refresh:{rt} + SREM user_sessions:{uid}
```

### 1.3 对比 JWT

| 维度 | JWT | Opaque + Redis（本方案） |
|---|---|---|
| 撤销 | 维护黑名单（失去无状态优势） | DEL key 即时 |
| 信息泄露 | payload 可解码 | 随机串，无信息 |
| 改密下线 | 难 | 删 user_sessions 集合 |
| 多设备管理 | 难 | 自然支持 |
| 风控介入 | 难实时 | lookup 时加 hook |
| 服务端伸缩 | 完全无状态 | 需 Redis 集群（已有） |
| 性能 | 单签名验证 | Redis lookup（≤1ms）|

**淘宝/京东/抖音都走 Opaque + Session**（Chrome DevTools 抓 cookie 验证）。

---

## 2. 安全特性

| 特性 | 实现 |
|---|---|
| **CSRF 防护** | access_token 配 csrf_token；写操作 `X-CSRF-Token` header 与 session.csrfToken 比对（双重提交） |
| **限流** | 同 IP 5 分钟 5 次登录失败 → 锁 30 min；同手机号 5 次 → 锁 24h（Redis `SETEX` + `INCR`） |
| **滑块/人机** | 接极验/腾讯防水墙；高风险登录强制 + 二次短信 |
| **设备指纹** | 客户端采集 UA + 屏幕 + canvas + WebGL hash → SHA256 → deviceId；服务端绑 refresh_token |
| **异常登录** | 新设备 / 异地 IP → 短信通知 + 强制二次验证 |
| **HttpOnly Cookie**（Web PC）| `Set-Cookie: access_token=...; HttpOnly; Secure; SameSite=Lax` |
| **Refresh 旋转** | 每次 refresh 同时换新 refresh_token，旧的作废；rotateCount > 10 强制重登 |
| **设备列表** | 「我的设备」查 user_sessions + 显示 ip/deviceInfo/lastActive；可一键踢出 |
| **改密强制下线** | 删整个 user_sessions:{uid} 集合 |
| **风控拦截** | session 读取时调风控 RPC，命中黑名单立即 401 |

---

## 3. 多种登录方式

| 方式 | 优先级 | 备注 |
|---|---|---|
| **手机号 + 短信验证码** | P1 主流 | 接阿里云/腾讯云短信 SDK |
| **用户名 + 密码 + 滑块** | P0 兜底 | 必须配滑块 + 限流 |
| **微信扫码** | P2 | 接微信开放平台 OAuth2 |
| **支付宝扫码** | P2 | 支付宝 OAuth2 |
| **Apple ID** | P2 | 上 App Store 强制要求 |
| **一键登录**（运营商 SDK） | P2 | 极光 / 阿里云 PNS，APP 内最快 |
| **人脸 / 指纹** | P3 | APP 端，二次确认 |

---

## 4. 多端 SSO（远期）

```
passport.yw-mall.com         主域，所有登录入口
  ├─ www.yw-mall.com         C 端主站
  ├─ m.yw-mall.com           H5
  ├─ admin.yw-mall.com       后台
  └─ seller.yw-mall.com      店家
```

主域颁发 sso_ticket（短期一次性）；子域用 ticket 换自己域下 access_token。
一键登出 = 删主域 + 广播到所有子域。

---

## 5. 实施路径与时间预算

### **P0 — 替换 JWT，修好登录**

> **范围：** 把 JWT 完全替换成 opaque token + Redis session，让 user-rpc → mall-api → handler 全链路通；浏览器跑真实 login 拿 token，不再手工签。

| Story | 工时 | 主要文件 |
|---|---|---|
| **L0.1** mall-user-rpc 加 Session RPC: `CreateSession` / `ValidateSession` / `RefreshSession` / `DestroySession` / `DestroyAllUserSessions` | 2d | 新建 5 个 logic + redis 接入 svc |
| **L0.2** mall-api / mall-admin-api 鉴权中间件: `JwtMiddleware` → `SessionMiddleware`（调 user-rpc.ValidateSession 或直读 Redis） | 1d | middleware/auth.go |
| **L0.3** Login 端点改造: `/api/auth/login` 替代 `/api/user/login`，返回 `{accessToken, refreshToken, expiresIn, csrfToken}` | 1d | mall-api handler + logic |
| **L0.4** `/api/auth/refresh` / `/api/auth/logout` 端点 | 0.5d | 同上 |
| **L0.5** 老 `/api/user/login` 短期保留兜底返回新格式 | 0.5d | 兼容层 |
| **L0.6** yw-mall-fe + yw-mall-admin-fe: `request.ts` 拦截器自动 refresh，401 用 refresh_token 换新 | 1d | 两个仓库 |
| **L0.7** e2e + 浏览器 smoke | 1d | 测试 |

**P0 总计：7 人天 ≈ 1 周（1 后端 + 0.5 前端）**

**P0 范围内不做**：滑块、短信、微信扫码、设备指纹、SSO — 都是 P1+。

### **P1 — 多种登录方式 + 安全增强**

| Story | 工时 |
|---|---|
| L1.1 短信验证码登录（接阿里云/腾讯云 SMS） | 3d |
| L1.2 滑块/极验集成（人机校验） | 3d |
| L1.3 登录限流（IP + 手机号失败锁） | 1d |
| L1.4 CSRF 双重提交防护开关 | 1d |
| L1.5 「我的设备」管理页 + 踢出 | 3d |
| L1.6 改密后强制下线 | 1d |

**P1 总计：12 人天 ≈ 2.5 周**

### **P2 — 第三方登录 + 风控**

| Story | 工时 |
|---|---|
| L2.1 微信扫码 OAuth2 | 4d |
| L2.2 支付宝扫码 OAuth2 | 3d |
| L2.3 Apple ID（仅 APP）| 3d |
| L2.4 一键登录（极光/阿里云 PNS）| 3d |
| L2.5 设备指纹采集（前端 SDK + 后端入库 `user_device`） | 3d |
| L2.6 异常登录检测（异地 IP / 新设备 / 高频）+ 二次短信 | 4d |

**P2 总计：20 人天 ≈ 4 周**

### **P3 — SSO + 高级风控**

| Story | 工时 |
|---|---|
| L3.1 passport 子域 SSO 架构 | 5d |
| L3.2 子站点单点登出 | 3d |
| L3.3 接腾讯天御 / 阿里云风控引擎 | 5d |
| L3.4 行为分析（点击节奏、鼠标轨迹）| 5d |

**P3 总计：18 人天 ≈ 4 周**

### 累计总计

| 阶段 | 工时 | 目标 |
|---|---|---|
| P0 | 7d / 1 周 | 修好登录，浏览器跑真实账号 |
| P0+P1 | 19d / 3 周 | 短信 + 滑块 + 限流，接 50 家试运营店家底线 |
| P0+P1+P2 | 39d / 6 周 | 多种登录方式 + 风控，对外用户可用 |
| 全套 P0..P3 | 57d / 10 周 | 像淘宝/京东（不含国际化 + Apple/小程序） |

---

## 6. 与 MVP Sprint 的关系

| MVP Sprint | 关联建议 |
|---|---|
| **S4** 安全 / 实名 / 合规（W7-W8）| **P0 必须在此 Sprint 之前完成**（否则 S4 的 MFA / 实名 / 个保法接口都还在用坏的 JWT 跑） |
| S5 CI/CD | 自动化测试要覆盖新 login flow |
| S7-S8 店家 SPA | merchant 端登录走 P0 同一套（role=merchant，shopId 注入 session） |
| S11 真实支付 | 真实交易必须有限流 + 风控 = P1 必做 |
| S12 灰度上线 | P0+P1+P2 全部上线才能接外部用户 |

**推荐排期**：把 P0 插到 **S3 收尾 → S4 之间**，1 周搞定，下一个 sprint 开始时已经能用真实账号。

---

## 7. 实施跟踪

### 状态图例

| 状态 | 含义 |
|---|---|
| ⬜ | 未开始 |
| 🟦 | 进行中 |
| ✅ | 已完成 |
| ⚠️ | 进度风险 |
| 🚫 | 阻塞 |

### P0 跟踪表

| Story | 工时 | 状态 | 完成日 | 备注 |
|---|---|---|---|---|
| L0.1 Session RPC × 5 | 2d | ✅ | 2026-05-14 | CreateSession / ValidateSession / RefreshSession / DestroySession / DestroyAllUserSessions |
| L0.2 SessionMiddleware | 1d | ✅ | 2026-05-14 | mall-api/internal/middleware/session_auth.go；替换 30 个 logic 的 uid 提取 |
| L0.3 `/api/auth/login` | 1d | ✅ | 2026-05-14 | 返回 {accessToken, refreshToken, expiresIn, csrfToken} |
| L0.4 refresh / logout | 0.5d | ✅ | 2026-05-14 | refresh 旋转；logout DEL session+refresh |
| L0.5 老端点兼容层 | 0.5d | ✅ | 2026-05-14 | `/api/user/login` 内部调 CreateSession，返回新格式 |
| L0.6 前端 request.ts 拦截器 | 1d | ✅ | 2026-05-14 | yw-mall-fe：401 自动 refresh 单飞 + X-CSRF-Token；admin-fe 暂未改（沿用 JWT，下一波 P0.5）|
| L0.7 e2e + 浏览器 smoke | 1d | ✅ | 2026-05-14 | login/refresh/logout/401-after-logout 全通 |

### P1 跟踪表

| Story | 工时 | 状态 | 备注 |
|---|---|---|---|
| L1.1 短信登录 | 3d | ⬜ | 阿里云/腾讯云 SMS 资质 |
| L1.2 滑块/极验 | 3d | ⬜ | 极验商务对接 |
| L1.3 登录限流 | 1d | ⬜ | — |
| L1.4 CSRF | 1d | ⬜ | — |
| L1.5 我的设备 | 3d | ⬜ | — |
| L1.6 改密下线 | 1d | ⬜ | — |

### P2 / P3 跟踪表

（同上格式，先占位，待 P1 收尾后展开。）

---

## 8. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| Redis 集群不稳 → 全员登出 | 低 | 高 | Redis 主从已就位；监控 + Sentinel |
| Session lookup 增加延迟 | 低 | 中 | 单次 lookup < 1ms；连接池调优 |
| 短信通道资质审批 | 中 | 中 | P1 前提前申请 |
| 极验/防水墙商务对接 | 中 | 中 | P1 前同步启动 |
| 老 JWT token 在客户端缓存导致迁移期混乱 | 高 | 低 | L0.5 老端点保留 2 周；前端版本号自动失效 |
| 改密下线场景没覆盖到（如新功能引入新 token 类型） | 中 | 中 | 所有 token 一律通过 user_sessions 集合管理，新增类型必须加入索引 |

---

## 9. 下一步

| 决策点 | 选项 |
|---|---|
| 启动时机 | (a) S3 收尾后立即 P0；(b) S4 开始前夹在中间；(c) 与 S4 并行 |
| 团队投入 | (a) 1 后端 + 0.5 前端 = 1 周；(b) 2 后端并行 = 0.5 周（更快但代码冲突管理麻烦） |
| 资质准备 | P1 前必须确认短信通道账号 + 极验账号 |

---

## 10. 关联文档

- `planning/mvp_sprint_plan.md` — MVP 12 Sprint 计划
- `planning/q1q2_tracking.md` — Sprint Story 跟踪表
- `daily/2026-05-12-pm.md` — 首次发现 JWT 漂移 bug
- `daily/2026-05-14.md` — S3 收尾，再次确认 bug 仍存在

---

> **一句话总结**：P0 一周替换 JWT 为 opaque token + Redis session，让 yw-mall 立即获得"撤销/多设备/风控介入"三大能力；P1-P3 按 sprint 节奏并入 MVP 后续阶段；上线前的最小安全底线 = P0 + P1（3 周）。
