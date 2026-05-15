# Sprint 4 — 安全 / 实名 / 合规

**编写日期：** 2026-05-15
**状态：** 启动中
**关联：** `planning/mvp_sprint_plan.md` §7、`feat/login-revamp.md`（P0/P0.5 已收尾）
**目标：** 在 W7-W8 内把 8 个 Story 全部交付，外部资质未到的部分用 mock 兜底，
后续真接入只需换 SDK 调用层，业务/数据库/前端不动。

---

## 0. Sprint 4 设计原则

| 原则 | 说明 |
|---|---|
| **mock-first** | 短信/腾讯云 IDV 等外部依赖统一走 mock SDK，行为可配置（pass/fail/random）；接口契约对齐真实 SDK 文档 |
| **可观测** | 安全事件（失败登录 / 锁定 / MFA 启用 / KYC 提交 / 数据导出）全部入 op_log + Prometheus metric |
| **零回归** | P0/P0.5 登录链路保持不变，新增 MFA 是叠加层（用户没启用 MFA 行为完全等同 P0） |
| **配置驱动** | 失败锁阈值、密码策略最小长度、TOTP 验证窗口等全部可热改（`config_kv` 或 etcd） |
| **数据兼容** | 加密字段双写过渡：先读时尝试解密失败 fallback 明文；下个 sprint 跑迁移脚本 |

---

## 1. 8 个 Story 详细拆解

### S4.1 admin MFA（TOTP + 短信备用）— 3d

**触发动因**：admin token 一旦泄露相当于全权限，必须有第二层。

- **后端**（mall-user-rpc / mall-admin-api）
  - 新表 `admin_mfa { admin_id PK, totp_secret_enc, enabled, created_at, last_used_at }`
  - 新 RPC：`EnableAdminMfa(admin_id) -> {totp_secret, qr_url}` / `VerifyAdminMfa(admin_id, code) -> {ok}` / `DisableAdminMfa(admin_id, code)`
  - `AdminLogin` 流程扩展：密码通过后若 mfa.enabled=true，返回 `{mfaRequired:true, challengeToken:...}` 而非 access token；客户端再调 `/admin/v1/login/mfa` 带 challengeToken+code 换 session
  - 短信备用走 mock SMS：6 位数字存 Redis `sms:admin_mfa:{admin_id}`，TTL 5 min
  - 用 [`pquerna/otp/totp`](https://github.com/pquerna/otp) 实现 RFC 6238

- **admin-fe**
  - `pages/profile/mfa.vue`：扫码绑定、解绑、查看状态
  - 登录页：识别 `mfaRequired` → 弹 6 位验证码输入框

**验收**：admin 启用 MFA 后，密码正确但无 code → 401；密码 + 正确 code → 200。备用 SMS 通道 mock 后同样工作。

---

### S4.2 IP 白名单 + 失败 5 次锁 30 min — 2d

- **后端**（mall-user-rpc + mall-admin-api 中间件）
  - 新表 `admin_ip_whitelist { admin_id, cidr, note, create_time }`
    - 空白名单 = 不限制（避免初次部署锁死）
    - 任何 IP 不在白名单 → 401 + 写 op_log
  - Redis `login_fail:user:{username}` INCR + EXPIRE 30 min；`>= 5` 则 lock，登录直接 403 "账号已锁定，请 30 min 后重试"
  - Redis `login_fail:ip:{ip}` 同样规则，5 次锁 30 min（防扫号）
  - 成功登录后清零

- **admin-fe**
  - `pages/security/ip-whitelist.vue`：列表 + 新增 + 删除
  - 锁定状态在登录失败 toast 显示剩余分钟数

**验收**：6 次错密码后第 6 次直接 403；30 min 后自动恢复。白名单 CIDR 命中允许，不命中拒绝。

---

### S4.3 密码策略 — 2d

- **后端**（mall-user-rpc）
  - `CreatePassword` / `ChangePassword` 校验：≥8 字符、含大小写+数字（可选符号）
  - 新表 `password_history { user_id, role, hash, create_time }`，最多保留 5 条；新密码 hash 不能命中历史
  - 启动时加载策略到内存（`min_length / require_upper / require_digit / max_history / max_age_days`）— 写在 etcd，热改
  - `LastPasswordChangeAt + max_age_days < now()` 时返回 `{passwordExpired:true}`，强制弹改密页

- **mall-fe / admin-fe**
  - `pages/profile/change-password.vue`（C 端 + admin 各一）
  - 登录响应若带 `passwordExpired` → 自动跳改密页

**验收**：弱密码（"123456"）拒绝；改密时复用最近 5 个之一拒绝；超期后强制改密。

---

### S4.4 实名认证（OCR + 活体）— 5d（mock）

- **后端**（mall-user-rpc）
  - 新表 `user_kyc { user_id PK, status (0=未提交/1=审核中/2=通过/3=拒绝), real_name_enc, id_card_no_enc, id_card_front_url, id_card_back_url, face_video_url, reject_reason, submit_time, audit_time }`
  - 新 RPC：
    - `SubmitKyc(user_id, real_name, id_card_no, front_url, back_url, face_url) -> {request_id}` — 写库 status=1 + 异步触发 mock 审核（500ms 后随机 90% 通过）
    - `GetKycStatus(user_id) -> {status, reject_reason}`
    - `AdminAuditKyc(user_id, pass, reason)`（admin 兜底人工审核）
  - mock SDK 抽象：`KycProvider` 接口 + `MockKycProvider` 实现；后续 S11 时换 `TencentKycProvider`

- **mall-fe**
  - `pages/kyc/submit.vue`：上传 3 张图（前/后/面部）+ 输入姓名/身份证 → 提交
  - `pages/kyc/status.vue`：状态轮询 + 失败原因显示

- **admin-fe**
  - `pages/users/kyc-review.vue`：待审核列表 + 一键通过/拒绝 + 备注

**验收**：用户提交 → 状态变审核中 → mock 1s 后变通过 → 状态页显示"已认证"。

---

### S4.5 个保法数据查询/导出/删除接口 — 3d

- **后端**（mall-api 新路由组）
  - `GET /api/user/data/scope` — 列出本平台收集的 PII 字段清单（静态，从 yaml 读）
  - `POST /api/user/data/export` — 异步任务：读 user / user_address / user_kyc / order(receiver_*) → 拼 JSON → 写到 MinIO → 邮件下载链接（mock 邮件，写日志）
  - `POST /api/user/data/erase` — 软删除：user.status=2 + 关联表里 PII 字段 anonymize（"己注销用户"）；订单 receiver_* 保留作为账务依据但替换姓名/电话
  - 所有动作写 op_log（actor=用户自己 + 类型 = data_export/data_erase）

- **mall-fe**
  - `pages/profile/data.vue`：3 个按钮（查看清单 / 导出 / 注销账号）+ 注销前的红色确认 modal

**验收**：导出 JSON 包含所有 PII 字段；注销后该用户不能再登录。

---

### S4.6 敏感字段加密存储 — 3d

- **后端**（mall-common 加 `cryptox` 包）
  - `cryptox.Encrypt(plaintext) -> base64(nonce + ciphertext)` / `Decrypt(...)`
  - AES-256-GCM；密钥从 env `MALL_FIELD_ENCRYPTION_KEY` 读（32 字节 hex）；缺失时启动失败
  - 应用到字段：`user.phone` / `user_kyc.real_name` / `user_kyc.id_card_no` / `admin_mfa.totp_secret`
  - 读路径：先尝试解密，失败 fallback 明文（双写过渡，下个 sprint 跑迁移脚本批量加密老数据）
  - 写路径：永远加密

- **migration**
  - `scripts/migrate-encrypt-fields.sh`：扫表把存量明文 row 加密更新（idempotent，已加密的跳过）

**验收**：DB 直接看 phone 列是 base64；通过 RPC 读出来还是 138xxx；旧明文行不影响读取。

---

### S4.7 隐私 + 用户协议 + Cookie 提示 — 2d（前端）

- **mall-fe**
  - `pages/legal/privacy.vue` + `pages/legal/terms.vue`：纯静态 markdown 渲染（用 `marked` 库）
  - 协议内容写在 `src/static/legal/{privacy,terms}.md`
  - 首次访问检测：localStorage 无 `cookie_consent` → 底部弹 banner（"我们使用 Cookie..." + "同意"按钮 + "查看隐私政策"链接）
  - 登录页 + 注册页底部链接到这两个协议

**验收**：首次访问出 banner，点同意后 localStorage 标记，刷新不再出现。

---

### S4.8 OpLog 审计查询 UI — 2d

- **后端**（mall-admin-api）
  - 现有 OpLog middleware 当前只写 logx；改为额外 `INSERT INTO admin_op_log` （表已存在）
  - 新路由 `GET /admin/v1/op-log` 支持 filter：actor_id / actor_role / method / path / status / time range / 分页
  - 写入用 fire-and-forget goroutine，避免阻塞主请求

- **admin-fe**
  - `pages/security/op-log.vue`：表格 + 筛选 + 导出 CSV

**验收**：登录、改密、KYC 审核等动作都能在 UI 看到记录；筛选条件准确。

---

## 2. 时间预算

| Story | 工时 | Owner |
|---|---|---|
| S4.1 admin MFA | 3d | 后端 |
| S4.2 IP 白名单 + 失败锁 | 2d | 后端 |
| S4.3 密码策略 | 2d | 后端 |
| S4.4 实名认证（mock）| 5d | 后端 + 前端 |
| S4.5 个保法接口 | 3d | 后端 + 前端 |
| S4.6 敏感字段加密 | 3d | 后端 |
| S4.7 隐私 / 用户协议 / Cookie | 2d | 前端 |
| S4.8 OpLog 审计 UI | 2d | 后端 + 前端 |
| **合计** | **22d** | — |

按 1 后端 + 0.5 前端的配比：约 **3 周**。Sprint 4 标准是 2 周，所以 mock 简化下应能压到 2 周。

---

## 3. Story 跟踪表

| Story | 状态 | 完成日 | 备注 |
|---|---|---|---|
| S4.1 admin MFA | ⬜ | — | — |
| S4.2 IP 白名单 + 失败锁 | ⬜ | — | — |
| S4.3 密码策略 | ⬜ | — | — |
| S4.4 实名认证（mock）| ⬜ | — | — |
| S4.5 个保法接口 | ⬜ | — | — |
| S4.6 敏感字段加密 | ⬜ | — | — |
| S4.7 隐私 / 用户协议 / Cookie | ⬜ | — | — |
| S4.8 OpLog 审计 UI | ⬜ | — | — |

---

## 4. 风险 + 缓解

| 风险 | 缓解 |
|---|---|
| TOTP 时钟偏差 | 验证窗口 ±1 个 30s 周期；服务端 NTP 监控 |
| IP 白名单初次部署锁死 | 空表 = 不限制；admin 加 IP 时校验当前 IP 不会被自己排除 |
| 加密密钥丢失 = 数据丢失 | 启动时校验密钥能解密一条已知 ciphertext 的 canary row |
| 数据导出 / 注销不可逆 | 注销前红色 modal 二次确认；导出文件 7 天后清理 |
| mock IDV 被认为是真 KYC | 接口返回带 `mock:true` 字段；admin UI 显著标记 |
| OpLog 写库失败拖慢主请求 | fire-and-forget goroutine + 内部 ring buffer fallback |

---

## 5. 与 P0/P0.5 / Sprint 5 的关系

- **P0/P0.5 登录链路**：MFA 是叠加，未启用 MFA 的 admin 走原有流程
- **Sprint 5 CI/CD**：S4 完成后的功能必须 100% 进 e2e suite
- **Sprint 11 真实支付**：S4.4 KYC 必须接真实 SDK（这时换 KycProvider 实现即可）

---

## 6. 验收清单

- [ ] admin 启用 MFA 后 e2e 通过
- [ ] 6 次错密码触发锁定
- [ ] IP 不在白名单返回 401
- [ ] 弱密码 / 历史密码 拒绝
- [ ] KYC 提交 + 审核通过 + 状态查询全通
- [ ] 数据导出 JSON 包含所有 PII
- [ ] 数据注销后用户无法再登录
- [ ] DB 里 phone / real_name / id_card 为 base64
- [ ] 首次访问 cookie banner 出现
- [ ] op-log 查询 UI 显示当日所有写操作
- [ ] OWASP top 10 自查通过
