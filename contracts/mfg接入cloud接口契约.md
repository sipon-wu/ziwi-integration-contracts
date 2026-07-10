# mfg.ziwi.cn 接入 cloud.ziwi.cn · 接口契约（云侧实测填写）

> 用途：mfg 团队在 cloud.ziwi.cn 部署后，按 `school接入cloud接口契约模板.md` 填写**云侧实测验证值**，供 school/ecms 团队统一接入。
> 版本：v1.1｜日期：2026-07-10
> 对齐基线：`ziwi_school/main` 的 `cloud运营与运维/multi-product-platform-integration.md` v0.2 统一基线稿（提交 819190a）
> 填写人身份：知微制造（mfg.ziwi.cn）团队 — 对接负责人
> 证据标注：✅ = 线上实测确认；📄 = cloud 源码/设计文档依据；💻 = mfg 代码已实现

---

## A. 鉴权对接

### A.1 JWK 公钥接口契约

| 项 | 值 |
|---|---|
| 端点 | `GET https://cloud.ziwi.cn/api/v1/auth/public-key` ✅ |
| 是否需要认证 | 否 |
| 返回 Content-Type | `application/json` ✅ |
| 响应体结构 | 带 `data` 外包裹：`{"data":{"keys":[{...}]}}`；接入方建议双解析兼容裸 `{"keys":[...]}` 与 `data.keys` 💻 |
| `kid` 命名规则 | `key_v1`（当前），轮换后新增 `key_v2` 等；按 `kid` 选钥 ✅ |
| 缓存 TTL 建议 | `1h`（与公钥轮换周期对齐） |
| 密钥轮换旧 `kid` 保留窗口 | 建议 24h（两 kid 并存过渡） |
| JWT 签名算法 | `RS256`（RSA 4096-bit）✅ |
| 公私钥位置（cloud 容器内） | `/app/app/core/keys/private_v1.pem` / `public_v1.pem` |

### A.2 JWT claims 权威 schema（v0.2 基线：字段名统一为 `products`）

| 位置 | 字段 | 类型 | 必填 | 含义 | 接入方使用 |
|---|---|---|---|---|---|
| header | `alg` | string | 是 | `RS256` ✅ | 校验算法 |
| header | `kid` | string | 是 | 公钥标识 | 选 JWK |
| header | `typ` | string | 否 | `JWT` | — |
| payload | `sub` | string(UUID) | 是 | 用户 UUID | 识别用户 |
| payload | `email` | string | 是 | 用户邮箱 | 匹配本地账号 |
| payload | `tenant_id` | string | 是 | 所属租户 | 多租户隔离 |
| payload | **`products`** | **array\<string\>** | 是 | **用户有权限的产品标识字符串数组，如 `["mfg","school"]`**（v0.2 统一基线：字符串数组，非对象数组） | 各平台判断可访问产品（mfg: `"mfg" in cloud_products`）💻 |
| payload | `features` | array\<string\> | 过渡期 | **过渡期 cloud 双发，值同 `products`**（兼容旧消费方）；mfg 不依赖此字段 📄 | 过渡兼容（历史曾作模块清单，v0.2 起值=products） |
| payload | `iat` | int(unix) | 是 | 签发时间 | — |
| payload | `exp` | int(unix) | 是 | 过期时间 | 过期校验 |

> ⚠️ **字符串数组，非对象数组**：v0.2 顶部基线已明确 `products` 为字符串数组（如 `["school","mfg"]`），school 代码 `hasProduct(...,"school")` 与 mfg `payload.get("products",[])` 均按字符串数组处理（已实测）。原 v0.1 §3.3 的「对象数组（含 roles/license_exp）」写法已作废 —— 但 v0.2 文档 §3.3 示例仍残留对象数组，**以本契约与顶部基线声明为准，详见文末 §H 待确认项**。

> 完整示例 JWT（实测 decoded 语义，脱敏）：
> ```json
> {
>   "header": {"alg": "RS256", "kid": "key_v1", "typ": "JWT"},
>   "payload": {
>     "sub": "uuid-of-user",
>     "email": "user@example.com",
>     "tenant_id": "mfg_demo",
>     "products": ["mfg", "school"],
>     "features": ["mfg", "school"],
>     "iat": 1712345678,
>     "exp": 1712349278
>   }
> }
> ```

### A.3 登录重定向与 JWT 回传

| 项 | 值 |
|---|---|
| 接入方跳转 cloud 登录 URL | `https://cloud.ziwi.cn/login?redirect_uri=<回调URL>` |
| 入参 | `redirect_uri`（登录成功后回调）、`client_id`（可选） |
| JWT 回传方式 | URL query 参数 `?token=<JWT>` |
| 回传参数名 | `token` |
| 前端存储位置 | 建议 `localStorage` |

### A.4 Token 刷新契约

| 项 | 值 |
|---|---|
| 刷新端点 | `POST https://cloud.ziwi.cn/api/v1/auth/refresh` ✅ |
| 请求体 | `{"refresh_token": "<token>"}` ✅ |
| 响应体 | `{"access_token": "...", "refresh_token": "..."}` ✅ |
| access_token 有效期 | `1h`（3600s）📄 |
| refresh_token 有效期 | `7d`📄 |
| 旋转策略 | RFC 9700：刷新时旧 token 标记 `used` 发新 token；重放检测 → 撤销整条 family 📄 |

---

## B. 用户 / 租户匹配

### B.1 用户相关 API

| 端点 | 方法 | 认证 | 响应 |
|---|---|---|---|
| `/api/v1/auth/me` | GET | Bearer | `{"data":{"id":"uuid","email":"...","tenant_id":"mfg_xxx","products":[...]}}` ✅ |
| `/api/v1/auth/change-password` | POST | Bearer | `{"data":{"message":"..."}}` |
| `/api/v1/auth/logout` | POST | Bearer | `{"data":{"message":"..."}}` |

### B.2 email 匹配约定
- 接入方按 JWT `email` 匹配本地 `User` 表。
- 首次无匹配时：可自动创建用户（绑定 `cloud_uuid`）或提示管理员手动绑定。
- mfg 已实现 `users.cloud_uuid VARCHAR(36) UNIQUE` 关联 cloud 用户。
- email 建议统一归一化为小写。

### B.3 tenant_id 全局命名规范

| 项 | 值 |
|---|---|
| 规范 | `<产品前缀>_<客户代号>`，小写+数字+下划线，≤40 字符 |
| 产品前缀 | `mfg_`（制造）、`sch_`（教育）、`ai_`（AI赋能）、`fin_`（财税）、`ecms_`（能碳） |
| 示例 | `mfg_demo`、`mfg_acme_factory`、`sch_spring_primary` |
| 校验 | cloud 注册时校验前缀+格式，非法拒绝 |

---

## C. 授权校验语义

| 项 | 值 |
|---|---|
| `products` 含义 | 产品标识字符串数组（如 `["mfg","school"]`），来自 cloud `users.products`；判断「该用户能否访问本产品」用 `"mfg" in products` 💻 |
| `feature_flags`（模块级权限） | **独立于 JWT**，从各产品本地 `tenants.package_modules` (JSONB) 读取并扁平化（如 `{"M01":["WORK_ORDER"]}` → `{"M01_WORK_ORDER": True}`）💻；不塞进 JWT products |
| 用户在某产品内的角色（如 tenant_admin） | 走各产品本地租户角色体系（mfg 已有），不依赖 JWT products |
| License 到期日 | 走 License 服务/DB，不塞进 JWT |
| 403 错误体 | `{"detail":"Forbidden"}` |
| 当前 License 状态 | Phase 1 无硬 License 校验 |

---

## D. 私有部署心跳（v0.2 统一：每 1h 上报 / 连续 24h 失联）

| 项 | 值 |
|---|---|
| 心跳端点 | `POST https://heartbeat.ziwi.cn/api/v1/heartbeat` ✅ |
| 请求头 | `X-Api-Key`（cloud 签发）📄 |
| 上报频率 | **每 1 小时上报一次**（crontab / 定时任务）📄（v0.2 与《补充需求书合集》§6.2 对齐） |
| 失败重试 | 连续失败重试 3 次，间隔 5 分钟 |
| 失联判定 | **连续 24 小时无成功心跳 → 标记失联 + 标红 + 告警邮件，并进入降级模式**（限制新建/云端能力，已有内容只读；心跳恢复后自动解除）📄 |
| 到期提醒 | 到期前 30 天 / 7 天站内通知 |
| 私有部署认证 | 运营用户完全本地 IdP；管理员+财务员走 cloud；离线 License（内嵌 cloud 公钥） |
| ⚠️ 服务端配置待对齐 | heartbeat.ziwi.cn 当前实现为 `heartbeat_timeout_minutes=15` + `offline_threshold_misses=3` + `check_interval=5`（≈45min 判失联），**与 1h/24h 基线不符**，需改配置（见 §H） |

---

## E. 基础设施

| 项 | 值 |
|---|---|
| cloud 生产 URL | `https://cloud.ziwi.cn` ✅ |
| heartbeat URL | `https://heartbeat.ziwi.cn` ✅ |
| 服务器 | 193.112.163.147（腾讯轻量云） |
| TLS 证书 | Let's Encrypt `*.ziwi.cn` 通配符，有效期至 2026-09-25 |
| `/health` | `GET https://cloud.ziwi.cn/health` ✅ |

---

## F. 错误码总表

| HTTP 码 | 含义 | 接入方处理 |
|---|---|---|
| 200 | 成功 | — |
| 400 | 请求参数错误 | 检查请求体 |
| 401 | 未认证 / JWT 失效 / 签名错误 | 跳 cloud 登录 |
| 403 | 无权限 | 提示无权限 |
| 404 | 资源不存在 | — |
| 409 | 冲突（如邮箱已注册） | 提示错误 |
| 422 | 请求体验证失败 | 检查字段 |
| 429 | 限流 | 退避重试 |
| 500 | 服务端错误 | 提示稍后重试 |

---

## G. 交付确认清单

- [x] A.1 JWKS 端点已验证并返 RSA 公钥
- [x] A.2 JWT claims 已对齐 v0.2：`products` 字符串数组为主、`features` 过渡双发
- [x] A.3 登录重定向机制已定义
- [x] A.4 Token 刷新已实现（RFC 9700）
- [x] B.1 用户 API 已填写
- [x] B.2 email 匹配约定已定义
- [x] B.3 tenant_id 命名规范已定义（5 类前缀）
- [x] C 授权语义已填写（products 与 feature_flags 职责分离）
- [x] D 心跳基线已对齐 1h/24h（服务端配置待改，见 §H）
- [x] E 基础设施已填写
- [x] F 错误码已定义
- [x] v0.2 `features` vs `products` 字段差异已统一（products 为主）
- [x] 心跳频率/失联判定已与《补充需求书合集》§6.2 对齐（1h / 24h）

---

## H. 待协调方确认 / 已知问题（mfg 评审 v0.2 反馈）

| # | 级别 | 问题 | 建议处置 | 归属 |
|---|:----:|:-----|:---------|:-----|
| H1 | **P0** | **v0.2 文档 §3.3 仍画成对象数组（带 roles/license_exp），与顶部基线声明「字符串数组」直接冲突**。school 代码 `hasProduct(...,"school")` 与 mfg `payload.get("products",[])` 均按字符串数组处理（已实测）。§3.3 不修正，三方实现会分裂。 | coordinator 将 §3.3 示例改为字符串数组 `["mfg","school"]`；删除 roles/license_exp 嵌套（其语义见 H2） | ziwi_school 文档 |
| H2 | **P0** | **`roles`/`license_exp` 在字符串数组模型里无处安放**。§3.3 把角色/license 到期画进 products 元素，但 v0.2 决议是字符串数组。需明确二者来源。 | 明确：角色走各产品本地租户角色体系；license_exp 走 License 服务/DB；均不塞 JWT。基线应明示或删除 §3.3 嵌套字段 | ziwi_school 文档 |
| H3 | P3 | **v0.2 理由写反**：写「cloud 实际注入为 features，导致 mfg 登录被拒绝」。事实：cloud 一直注入 `products`（`jwt_service.py:25` 实测）；历史不一致是 **mfg 旧代码读 `features`**。mfg 迁移后已读 `products`，故「mfg 切到 products」动作其实已完成，无需新排期。 | 更正理由措辞，避免误导后续审计 | ziwi_school 文档 |
| H4 | P1 | **mfg 现状已与 v0.2 目标一致**：`cloud_products = payload.get("products",[])` + `feature_flags` 取自 DB `tenants.package_modules`；与 v0.2（products 字符串数组 + feature_flags 独立）一致。cloud 双发 `features` 对 mfg 非必需（mfg 已不读 JWT features）。`cloud_products` 目前 mfg 下游未实际消费，后续如需「无 mfg 产品权限则拒访」需补消费逻辑（mfg backlog，不影响基线）。 | 知悉即可；mfg 无需为 v0.2 排新代码改动 | mfg |
| H5 | P1 | **心跳服务端阈值与 1h/24h 基线不符**：heartbeat.ziwi.cn 当前 `timeout=15min`+`misses=3`+`check=5min`≈45min 判失联；若客户端按 1h 上报，服务端 45min 即误判失联。 | 改为匹配 1h/24h：建议 `check_interval=60`、`heartbeat_timeout_minutes=60`、`offline_threshold_misses=24`（或 `timeout=1440, misses=1`）。属心跳服务端小配置变更（非 JWT 基线，但涉及代码，需排 backlog） | heartbeat 服务 |

> 说明：H1/H2/H3 为 ziwi_school 文档侧修正项，H4 为 mfg 现状说明，H5 为心跳服务端配置待办。本契约以 v0.2 顶部基线声明（字符串数组）为权威，H1 修正前请以本契约 §A.2 为准。

---

## 参考：cloud API 清单

| 端点 | 方法 | 说明 | 认证 |
|---|---|---|---|
| `/api/v1/auth/register` | POST | 注册 | 否 |
| `/api/v1/auth/login` | POST | 登录签发 JWT | 否 |
| `/api/v1/auth/refresh` | POST | 刷新 access_token | 否 |
| `/api/v1/auth/me` | GET | 当前用户信息 | 是 |
| `/api/v1/auth/logout` | POST | 登出 | 是 |
| `/api/v1/auth/change-password` | POST | 修改密码 | 是 |
| `/api/v1/auth/public-key` | GET | RSA 公钥 JWK | 否 |
| `/health` | GET | 健康检查 | 否 |
