# mfg.ziwi.cn 接入 cloud.ziwi.cn · 接口契约（云侧实测填写）

> 用途：mfg 团队在 cloud.ziwi.cn 部署后，按 `school接入cloud接口契约模板.md` 填写**云侧实测验证值**，供 school/ecms 团队统一接入。
> 版本：v1.0｜日期：2026-07-10
> 填写人身份：知微制造（mfg.ziwi.cn）团队 — 对接负责人
> 证据标注：✅ = 线上实测确认；📄 = cloud 源码/设计文档依据；💻 = mfg 代码已实现。

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

### A.2 JWT claims 权威 schema

| 位置 | 字段 | 类型 | 必填 | 含义 | 接入方使用 |
|---|---|---|---|---|---|
| header | `alg` | string | 是 | `RS256` ✅ | 校验算法 |
| header | `kid` | string | 是 | 公钥标识 | 选 JWK |
| header | `typ` | string | 否 | `JWT` | — |
| payload | `sub` | string(UUID) | 是 | 用户 UUID | 识别用户 |
| payload | `email` | string | 是 | 用户邮箱 | 匹配本地账号 |
| payload | `tenant_id` | string | 是 | 所属租户 | 多租户隔离 |
| payload | `features` | array | 是 | 功能模块清单（从 `tenants.package_modules` 读取） | 控制模块可见性 |
| payload | `iat` | int(unix) | 是 | 签发时间 | — |
| payload | `exp` | int(unix) | 是 | 过期时间 | 过期校验 |

> 完整示例 JWT（实测 decoded，脱敏）：
> ```json
> {
>   "header": {"alg": "RS256", "kid": "key_v1", "typ": "JWT"},
>   "payload": {
>     "sub": "uuid-of-user",
>     "email": "user@example.com",
>     "tenant_id": "mfg_demo",
>     "features": ["M01","M02","M03","M04","M05","M06","M07","M08","M09","M10","M11","M12","M13"],
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
| `/api/v1/auth/me` | GET | Bearer | `{"data":{"id":"uuid","email":"...","tenant_id":"mfg_xxx","features":[...]}}` ✅ |
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
| `features` 含义 | 从 `tenants.package_modules` (JSONB) 读取，如 `["M01","M02",...]` |
| 403 错误体 | `{"detail":"Forbidden"}` |
| 当前 License 状态 | Phase 1 无硬 License 校验 |

---

## D. 私有部署心跳

| 项 | 值 |
|---|---|
| 心跳端点 | `POST https://heartbeat.ziwi.cn/api/v1/heartbeat` ✅ |
| 请求头 | `X-Api-Key`（cloud 签发）📄 |
| 离线检测 | 连续 3 次（15min）未上报 → WARN |
| License 到期检测 | 30d → WARN；7d → CRITICAL 📄 |
| 私有部署认证 | 运营用户完全本地 IdP；管理员+财务员走 cloud；离线 License（内嵌 cloud 公钥） |

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
- [x] A.2 JWT claims 已按 cloud `create_access_token` 实现填写
- [x] A.3 登录重定向机制已定义
- [x] A.4 Token 刷新已实现（RFC 9700）
- [x] B.1 用户 API 已填写
- [x] B.2 email 匹配约定已定义
- [x] B.3 tenant_id 命名规范已定义（5 类前缀）
- [x] C 授权语义已填写
- [x] D 心跳已部署并验证
- [x] E 基础设施已填写
- [x] F 错误码已定义
- [ ] 与 school 团确认 `features` vs `products` 字段差异，统一接入方校验逻辑
- [ ] 心跳频率/失联判定冲突待与 school 团拍板

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
