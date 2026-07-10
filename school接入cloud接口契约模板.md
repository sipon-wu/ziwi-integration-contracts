# school.ziwi.cn 接入 cloud.ziwi.cn · 接口契约模板

> 用途：mfg.ziwi.cn 团队部署 cloud.ziwi.cn 后，按本模板填写**实测验证值**，交付 school 团队据此实现 `verify_cloud_token` 中间件与对接逻辑。
> 版本：v1.1（school 团队实测填写）｜日期：2026-07-10
> 填写人身份：知微教学（school.ziwi.cn）团队 — 对接负责人（本对话 agent 代填）｜填写依据：①线上实测（heartbeat.ziwi.cn 健康检查/路由探测，2026-07-10，只读、未发任何写请求）②《技术文档》§12.3 心跳机制设计稿 ③ ziwi_school 代码（config.go / internal/cloud/jwks.go / auth_cloud.go）。
> 证据标注：✅ = 线上实测确认；📄 = 设计文档依据（待 cloud/mfg 用云源码回确认）；💻 = ziwi_school 代码证据。标注「⚠️冲突」项为文档间不一致、待拍板。
> 依据：`multi-product-platform-integration.md` v0.1（讨论稿，本模板为其"已部署验证"落地版）
> 关联：`产品规划/账户系统与cloud.ziwi.cn对接方案.md` §3.1–§3.8（school 侧对接契约）

---

## 填写说明

- 本模板所有 `[待填：…]` 项由 **mfg 团队在 cloud 部署完成后实测填写**，不得直接抄 v0.1 讨论稿占位。
- school 团队**仅依赖本文件**实现接入，不依赖 v0.1 草案的推测值。
- 任何与 `multi-product-platform-integration.md` v0.1 不一致之处，以本文件实测值为准，并回写 cloud 主文档。

---

## A. 鉴权对接（school 实现 `verify_cloud_token` 中间件所需）

### A.1 JWK 公钥接口契约

| 项 | 值 |
|---|---|
| 端点 | `GET https://cloud.ziwi.cn/api/v1/auth/public-key` 💻（school `config.go` 默认值 `CLOUD_JWKS_URL`，已线上证书有效） |
| 是否需要认证 | 否 |
| 返回 Content-Type | `application/json` 💻（school `jwks.go` 按 JSON 解析） |
| 响应体结构（示例，按实测填写） | 带 `data` 外包裹：`{"data":{"keys":[{kid,kty,alg,n,e}…]}}`；school 端**双解析**兼容裸 `{"keys":[...]}` 与 `data.keys` 💻（线上实测踩坑，见 school 对接方案 §7.1） |
| `kid` 命名规则 | 待云源码确认（school 按 `kid` 选钥、不写死单把公钥）💻 |
| school 端缓存 TTL 建议 | `1h`（school `jwks.go` TTL 1h，与 cloud 公钥轮换周期对齐）💻 |
| 密钥轮换时旧 `kid` 保留窗口 | 待云源码确认（school 支持 `kid` 平滑轮换，两 kid 并存期由 cloud 决定）💻 |

> 注意：school 必须按 `kid` 选钥（支持 key_v1→key_v2 平滑轮换），不能写死单把公钥。

### A.2 JWT claims 权威 schema

| 位置 | 字段 | 类型 | 必填 | 含义 | school 使用方 |
|---|---|---|---|---|---|
| header | `alg` | string | 是 | `[待填：RS256]` | 校验算法 |
| header | `kid` | string | 是 | 公钥标识 | 选 JWK |
| header | `typ` | string | 否 | `[待填：JWT]` | — |
| payload | `sub` | string(UUID) | 是 | 用户 UUID | 识别用户身份 |
| payload | `email` | string | 是 | 用户邮箱 | 匹配 school 已有账号 |
| payload | `tenant_id` | string | 是 | 所属租户 | 多租户隔离 |
| payload | `products` | array | 是 | 有权限产品列表 | 判断可访问功能 |
| payload | `products[].id` | string | 是 | 产品标识（如 `school`） | 判断本产品权限 |
| payload | `products[].roles` | array | 是 | 该产品内角色 | 权限判断 |
| payload | `products[].license_exp` | string(date) | 是 | License 到期日 | License 校验 |
| payload | `iat` | int(unix) | 是 | 签发时间 | — |
| payload | `exp` | int(unix) | 是 | 过期时间 | 过期校验 |
| payload | 其他字段 | — | — | `[待填：如有额外 claim 请列明]` | — |

> 完整示例 JWT（粘贴实测签发的一例，脱敏）：
> ```
> [待填：粘贴一例真实 JWT 的 decoded header/payload]
> ```

### A.3 登录重定向与 JWT 回传机制

| 项 | 值 |
|---|---|
| school 跳转 cloud 登录的 URL | `[待填：如 https://cloud.ziwi.cn/login?redirect_uri=...]` |
| 入参（redirect_uri / client_id 等） | `[待填：参数名与含义]` |
| 登录成功后 JWT 回传方式 | `[待填：query 参数名 / fragment / 其他]` |
| 回传参数名 | `[待填：如 token=]` |
| 前端存储位置 | `[待填：localStorage / 内存；v0.1 §5.4 建议内存或 localStorage]` |
| 回调落地页 | `[待填：school 前端哪个路由接收]` |

### A.4 Token 刷新契约

| 项 | 值 |
|---|---|
| 刷新端点 | `POST [待填：如 https://cloud.ziwi.cn/api/v1/auth/refresh]` |
| 请求体 | `[待填：如 {refresh_token: ...}]` |
| 响应体 | `[待填：如 {access_token, refresh_token}]` |
| access_token 有效期 | `[待填：如 30min]` |
| refresh_token 有效期 | `[待填：如 7d]` |
| refresh_token 存储位置 | `[待填：localStorage / httpOnly cookie]` |
| 是否需要认证 | 否（需 refresh_token） |

---

## B. 用户 / 租户匹配

### B.1 用户相关 API

| 端点 | 方法 | 认证 | 请求 | 响应（实测 schema） |
|---|---|---|---|---|
| `/api/v1/auth/me` | GET | Bearer | — | `[待填：粘贴真实响应]` |
| `/api/v1/users/{id}` | GET | Bearer | — | `[待填]` |
| `/api/v1/users/{id}/products` | PATCH | Bearer | `[待填：如 {products:[...]}]` | `[待填]` |

### B.2 email 匹配约定
- school 按 JWT `email` 匹配本地 `User` 表；无匹配时由管理员手动绑定 `cloud_user_id`（见 school 对接方案 §3.2）。
- `[待填：如 email 大小写是否归一化、是否允许改绑，请明确]`

### B.3 tenant_id 全局命名规范
- `[待填：全局唯一命名规则，如 <产品前缀>_<客户代号>，示例 acme_factory；需 mfg 与 school 共用同一规范防撞车]`

---

## C. 授权校验语义

| 项 | 值 |
|---|---|
| 无本产品 License 时的行为 | `[待填：如返回 403，提示"无此产品 License"（v0.1 §5.3）]` |
| 403 错误体格式 | `[待填：粘贴真实错误 JSON]` |
| `license_exp` 过期处理 | `[待填：school 本地兜底 + cloud JWT 优先，见对接方案 §3.5]` |
| `roles` 与 school RoleMatrix 映射 | `[待填：products[].roles 取值集合，如 tenant_admin/teacher/...]` |

---

## D. 私有部署心跳（对应 school 对接方案 §3.8）

| 项 | 值 |
|---|---|
| 心跳端点 | `https://heartbeat.ziwi.cn/api/v1/heartbeat` ✅（线上实测 2026-07-10：`GET` 返回 **405 Method Not Allowed / allow: POST**，证明路由已部署；`/health` 返回 200；DNS→`193.112.163.147`；TLS 证书 Let's Encrypt 有效至 2026-09-25） |
| 请求方法 | `POST`（仅 POST，GET 返回 405）✅ |
| 请求头 | `X-School-ID`（学校标识）、`X-Signature`（RSA-SHA256 签名）、`X-Nonce`（随机值，防重放）、`X-Timestamp`（unix 秒，±5min 窗口）📄（依据《技术文档》§12.3；字段名大小写待云源码回确认） |
| 签名算法 | `RSA-SHA256(school_id + token_used + nonce + timestamp)` 📄（§12.3） |
| 请求体 schema | `{ "school_id": string, "token_used": int, ... }` 随签名一并发送 📄（§12.3；完整字段集待云源码确认） |
| 上报频率 | ⚠️ **冲突待拍板**：《技术文档》§12.3 写「每 1 小时」；本模板原稿与 school 对接方案 §3.8 写「每天 1 次」。需 cloud/mfg 团队定稿。 |
| 云端校验 | Timestamp ±5min 防重放；Nonce 10min 内未使用（Redis 去重）；签名验证通过；License 未过期/未吊销 📄（§12.3） |
| 失联判定 | ⚠️ **冲突待拍板**：§12.3 写「连续 24 小时失败→降级模式」；school 对接方案 §3.8 写「连续 3 天未上报标记失联」。需定稿。 |
| 重试策略 | 指数退避 1→2→4→8→16 min 📄（§12.3） |
| 该端点是否已随 cloud 部署 | `是` ✅（实测路由已响应 405；vhost 与证书已就位，无需 school 侧独立代码，复用 cloud 同服务 `/api/v1/heartbeat` 反代） |

---

## E. 基础设施

| 项 | 值 |
|---|---|
| cloud 生产 base URL | `https://cloud.ziwi.cn` ✅（config 默认值 + 线上证书有效） |
| cloud 预发布 base URL | 待确认（school 侧暂未配置 staging cloud 地址） |
| CORS 允许 origin 列表 | 待云源码确认（school 前端 `api.ts` 直连 `cloud.ziwi.cn`，需 CORS 放行 `school.ziwi.cn` / `school1.ziwi.cn`） |
| `/health` 端点 | `https://heartbeat.ziwi.cn/health` 已实测 200 ✅；`https://cloud.ziwi.cn/health` 待实测 |
| 版本号端点 | 待确认 |

---

## F. 错误码总表

| HTTP 码 | 含义 | school 前端处理 |
|---|---|---|
| 401 | `[待填：未认证/JWT 失效]` | 跳 cloud 登录（对接方案 §3.1） |
| 403 | `[待填：无 License/无权限]` | 提示无权限 |
| 402 | `[待填：Token 不足（如有）]` | 现有弹窗逻辑（api.ts） |
| 429 | `[待填：限流]` | 现有处理 |
| 其他 | `[待填]` | — |

---

## G. 交付确认清单（mfg 团队部署后勾选）

- [x] A.1 JWKS 端点/结构/缓存 已由 school 代码证据填写（💻），kid 命名/轮换窗口待云源码
- [ ] A.2–A.4 待 cloud 提供真实 JWT 示例与刷新契约（school 代码已支持 RS256 验签，可据此实现）
- [ ] B.1–B.3 待 cloud 提供实测响应与 tenant_id 规范
- [ ] C 授权语义与错误体待 cloud 提供
- [x] D 心跳端点已线上实测确认存在并已填（✅）；频率/失联判定两项文档冲突待拍板
- [x] E 基础设施 base URL / heartbeat health 已填（✅），CORS 待云源码确认
- [ ] F 错误码总表待 cloud 提供
- [ ] 与 `multi-product-platform-integration.md` v0.1 不一致处（频率/失联判定/域名 license→heartbeat）已在本文件标「⚠️冲突」，待云源码回确认后回写 cloud 主文档
- [x] school 团队已收到本文件（v1.1）并据 D 段确认心跳端点可用，可进入 school 侧 P2 心跳客户端实现评估

---

## 参考：cloud 文档 v0.1 原有 API 清单（待上述实测覆盖）

| 端点 | 方法 | 说明 | 认证 |
|---|---|---|---|
| `/api/v1/auth/register` | POST | 注册 | 否 |
| `/api/v1/auth/login` | POST | 登录签发 JWT | 否 |
| `/api/v1/auth/refresh` | POST | 刷新 access_token | 否（需 refresh_token） |
| `/api/v1/auth/me` | GET | 当前用户 | 是 |
| `/api/v1/auth/public-key` | GET | RSA 公钥 JWK | 否 |
| `/api/v1/users/{id}` | GET | 查询用户 | 是 |
| `/api/v1/users/{id}/products` | PATCH | 更新 products | 是 |
| `/health` | GET | 健康检查 | 否 |
