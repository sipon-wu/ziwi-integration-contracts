# mfg.ziwi.cn 接入 cloud.ziwi.cn · 接口契约（云侧实测填写）

> 用途：mfg 团队在 cloud.ziwi.cn 部署后，按 `school接入cloud接口契约模板.md` 填写**云侧实测验证值**，供 school/ecms 团队统一接入。
> 版本：v1.2｜日期：2026-07-10
> 对齐基线：`ziwi_school/main` 的 `cloud运营与运维/multi-product-platform-integration.md` **v0.3 统一基线稿**（H1/H2/H3 提交 cc07072；H5 §5.5 同步提交 0c8ce44，均已合入 main）
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

### A.2 JWT claims 权威 schema（v0.3 基线：字段名统一为 `products` 字符串数组）

| 位置 | 字段 | 类型 | 必填 | 含义 | 接入方使用 |
|---|---|---|---|---|---|
| header | `alg` | string | 是 | `RS256` ✅ | 校验算法 |
| header | `kid` | string | 是 | 公钥标识 | 选 JWK |
| header | `typ` | string | 否 | `JWT` | — |
| payload | `sub` | string(UUID) | 是 | 用户 UUID | 识别用户 |
| payload | `email` | string | 是 | 用户邮箱 | 匹配本地账号 |
| payload | `tenant_id` | string | 是 | 所属租户 | 多租户隔离 |
| payload | **`products`** | **array\<string\>** | 是 | **用户有权限的产品标识字符串数组，如 `["mfg","school"]`**（v0.3 统一基线：字符串数组，非对象数组） | 各平台判断可访问产品（mfg: `"mfg" in cloud_products`）💻 |
| payload | `features` | array\<string\> | 过渡期 | **过渡期 cloud 双发，值同 `products`**（仅作旧消费方兼容，对 mfg 非必需）📄 | 过渡兼容 |
| payload | `iat` | int(unix) | 是 | 签发时间 | — |
| payload | `exp` | int(unix) | 是 | 过期时间 | 过期校验 |

> ✅ **v0.3 已闭环（H1/H2/H3）**：`products` 为字符串数组（如 `["mfg","school"]`），school `hasProduct(...,"school")` 与 mfg `payload.get("products",[])` 均按字符串数组处理（已实测）。`roles`（产品内角色，如 tenant_admin）走**各产品本地角色体系**；`license_exp`（License 到期）走 **License 服务/DB**；二者均**不进 JWT**（v0.3 §3.3 已明确，原对象数组写法作废）。

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

## D. 私有部署心跳（v0.3 统一：每 1h 上报 / 连续 24h 失联）

| 项 | 值 |
|---|---|
| 心跳端点 | `POST https://heartbeat.ziwi.cn/api/v1/heartbeat` ✅ |
| 请求头 | `X-Api-Key`（cloud 签发）📄 |
| 上报频率 | **每 1 小时上报一次**（crontab / 定时任务）📄（v0.3 与《补充需求书合集》§6.2 对齐） |
| 失败重试 | 连续失败重试 3 次，间隔 5 分钟 |
| 失联判定 | **连续 24 小时无成功心跳 → 标记失联 + 标红 + 告警邮件，并进入降级模式**（限制新建/云端能力，已有内容只读；心跳恢复后自动解除）📄 |
| 到期提醒 | 到期前 30 天 / 7 天站内通知 |
| 私有部署认证 | 运营用户完全本地 IdP；管理员+财务员走 cloud；离线 License（内嵌 cloud 公钥） |
| ✅ 服务端阈值（H5 已闭环） | heartbeat.ziwi.cn 已通过 `.env`+`docker-compose.yml` 注入 `HEARTBEAT_CHECK_INTERVAL_MINUTES=60` / `HEARTBEAT_TIMEOUT_MINUTES=60` / `HEARTBEAT_OFFLINE_THRESHOLD_MISSES=24`（纯 env 覆盖，未改源码），重启后容器内 env 与启动日志（`check_interval=60 min`）已核验生效；客户端按 1h 上报不再误判，连续 24 次失败≈24h 才标记失联，与基线一致 |

### D.1 心跳上报客户端 SDK（mfg 已实现）

> 位置：`code/heartbeat/client/`（与心跳服务端同仓 ziwi_mfg，低优先级模块补全）。本 SDK 供私有部署侧（如 mfg/school/ecms）一行挂载即可按 D 节基线自动上报心跳；**离线判定与告警仍由服务端负责**（见 D 节），客户端只管「每小时发一次」。💻✅

**代码位置与文件结构**

| 文件 | 说明 | 证据 |
|---|---|---|
| `client/__init__.py` | 导出 `HeartbeatClient`、`HeartbeatClientConfig`、`create_heartbeat_lifespan` | 💻 |
| `client/config.py` | `HeartbeatClientConfig(BaseSettings)`，env 前缀 `HEARTBEAT_` | 💻 |
| `client/heartbeat_client.py` | 核心 `HeartbeatClient`（调度 + 上报 + 重试） | 💻 |
| `client/fastapi_integration.py` | `create_heartbeat_lifespan()` | 💻 |
| `client/requirements.txt` | httpx / apscheduler / pydantic-settings | 💻 |
| `client/tests/` | 15 个用例（15/15 通过） | 💻✅ |
| `client/README.md` | 用法 + `HEARTBEAT_*` 配置清单 | 💻 |

**配置项（`HEARTBEAT_` 前缀，全部来自 env）**

| env 名 | 默认值 | 含义 |
|---|---|---|
| `HEARTBEAT_SERVER_URL` | `https://heartbeat.ziwi.cn` | 心跳服务端地址 |
| `HEARTBEAT_API_KEY` | （无默认，必填） | cloud 签发 API Key，用作请求头 `X-Api-Key` |
| `HEARTBEAT_DEPLOYMENT_ID` | （无默认，必填） | 部署实例唯一标识 |
| `HEARTBEAT_TENANT_ID` | （无默认，必填） | 所属租户 |
| `HEARTBEAT_PRODUCT` | （无默认，如 `mfg`） | 产品标识 |
| `HEARTBEAT_VERSION` | （无默认） | 部署版本号 |
| `HEARTBEAT_LICENSE_ISSUED_AT` | （无默认） | License 签发时间，**首次上报必带** |
| `HEARTBEAT_LICENSE_EXPIRES_AT` | （无默认） | License 到期时间，**首次上报必带** |
| `HEARTBEAT_INTERVAL_SECONDS` | `3600`（=1h） | 上报周期，对齐 D 节 1h 基线 |
| `HEARTBEAT_MAX_RETRIES` | `3` | 失败最大重试次数 |
| `HEARTBEAT_RETRY_BACKOFF_SECONDS` | `10` | 重试退避基数（指数退避） |
| `HEARTBEAT_REQUEST_TIMEOUT` | `10` | 单次请求超时（秒） |

**接入方式**

- FastAPI 一行挂载（推荐）：

```python
from heartbeat.client import create_heartbeat_lifespan
from fastapi import FastAPI

app = FastAPI(lifespan=create_heartbeat_lifespan())
```

- 最小独立用法：

```python
from heartbeat.client import HeartbeatClient, HeartbeatClientConfig

cfg = HeartbeatClientConfig()      # 读取 HEARTBEAT_* env
client = HeartbeatClient(cfg)
client.start()                     # 立即首发一次 + 调度周期上报（IntervalTrigger，默认 1h）
# ... 应用运行 ...
client.stop()
```

**与 D 节基线的对应关系**

| 维度 | 客户端职责 | 服务端职责 |
|---|---|---|
| 上报频率 | 每 `INTERVAL_SECONDS`（默认 3600s=1h）上报一次；`start()` 立即首发一次，调度用 `IntervalTrigger(seconds=interval_seconds)` 💻✅ | — |
| License 首报 | **首次上报必带 `license_issued_at` + `license_expires_at`**（缺失则 400）；未注册实例被 400 时自动补带 license 重试一次 💻✅ | 校验 license 字段，缺失返回 400 |
| 失败重试 | 按 `MAX_RETRIES` + 指数退避重试；**无论成败都不抛未捕获异常**，返回 `{"status":"error","detail":...}` + 日志告警（降级原则：实例照常运行）💻✅ | — |
| 离线判定 / 告警 | 不负责（连续 24h 无心跳才判 offline，由服务端算） | **连续 24h 无成功心跳 → 标失联 + 告警 + 降级**（见 D 节）📄 |
| 生命周期 | `start()`/`stop()` 幂等；`create_heartbeat_lifespan()` 可一行挂进 FastAPI 💻✅ | — |

> ✅ **验证**：`client/tests/` 15/15 用例通过；真实 `BackgroundScheduler` 实证周期心跳多次触发（5 秒窗口内触发 5 次）；QA 独立 Round 2 回归 `IS_PASS=YES`。

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
- [x] A.2 JWT claims 已对齐 v0.3：`products` 字符串数组为主、`features` 过渡双发
- [x] A.3 登录重定向机制已定义
- [x] A.4 Token 刷新已实现（RFC 9700）
- [x] B.1 用户 API 已填写
- [x] B.2 email 匹配约定已定义
- [x] B.3 tenant_id 命名规范已定义（5 类前缀）
- [x] C 授权语义已填写（products 与 feature_flags 职责分离）
- [x] D 心跳基线已对齐 1h/24h，服务端阈值已修复并核验（H5）
- [x] E 基础设施已填写
- [x] F 错误码已定义
- [x] v0.2 `features` vs `products` 字段差异已统一（products 为主，v0.3 闭环）
- [x] 心跳频率/失联判定已与《补充需求书合集》§6.2 对齐（1h / 24h，v0.3 闭环）
- [x] D.1 心跳上报客户端 SDK 已实现并验证（code/heartbeat/client/，15/15 测试通过，真实调度器周期触发已实证）💻✅

---

## H. mfg 评审 v0.2 → v0.3 闭环记录

| # | 级别 | 问题 | 处置结果 | 归属 |
|---|:----:|:-----|:---------|:-----|
| H1 | P0 | v0.2 §3.3 仍是对象数组，与顶部基线「字符串数组」冲突 | ✅ **已闭环**：v0.3 §3.3 改为字符串数组 `["mfg","school"]`（cc07072） | ziwi_school 文档 |
| H2 | P0 | `roles`/`license_exp` 在字符串数组模型无处安放 | ✅ **已闭环**：v0.3 明确角色走本地、license 走 License 服务/DB，均不进 JWT（cc07072） | ziwi_school 文档 |
| H3 | P3 | v0.2 理由写反（称 cloud 注入 features） | ✅ **已闭环**：v0.3 措辞纠正——cloud 始终注入 `products`（jwt_service.py:25 实测），从未注入 features（cc07072） | ziwi_school 文档 |
| H4 | P1 | mfg 是否已读 products / 双发 features 是否必需 | ✅ **已闭环（知悉）**：mfg 当前 `payload.get("products",[])` + `feature_flags` 取自 DB，与 v0.3 一致；cloud 保留双发 `features` 仅作旧消费方兼容 | mfg / cloud |
| H5 | P1 | 心跳服务端阈值（≈45min）与 1h/24h 基线不符 | ✅ **已闭环**：服务端 env 改为 `check=60`/`timeout=60`/`misses=24`，重启核验生效，§5.5 标注已修复（0c8ce44，已合 main） | heartbeat 服务 |

> 状态：5 项全部闭环（2026-07-10）。共享库文档现以 **v0.3（ziwi_school/main）** 为权威基线，各方以 v0.3 为准。

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
