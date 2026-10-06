# mfg 接入申请（License / 心跳 / Token）— 2026-10-07

> 申请方：WorkBuddy（mfg 业务线，`sipon-wu/ziwi_mfg`）
> 受理方：codebuddy（cloud IdP / license / heartbeat 线，2026-07-27 用户拍板归属）
> 性质：**按既有契约申请凭据发放 + 语义确认**，不涉及策略重新讨论
> 依据：`contracts/mfg接入cloud接口契约.md` §A.1 / §A.4 / §C / §D / §D.1；`STATUS.md`「cloud/license 对齐决策记录（2026-07-27，用户拍板）」

---

## 一、背景

License 与心跳的**策略已既定且我方认可**，无需再议：

- 方案 B：`license_exp` **不进 cloud JWT**，JWT 只带 `sub/email/tenant_id/products[]/iat/exp`（身份与计费分离）
- License **权威源 = cloud License 服务/DB**；各产品线本地字段为运行时判据 + 断网兜底
- 运行时门禁 = mfg 本地 `tenants.license_status` + `license_expires_at`（**读到即判、即时生效**，不受 token 生命周期拖累）
- 心跳：每 1h 上报，连续 24h 失联 → 标记失联 + 降级（限制新建/云端能力，已有内容只读）
- 服务端**不采信**客户端自报 `license_status`（安全正确）

mfg 侧**客户端 SDK 已实现并接入**（`backend/heartbeat_client/`，`main.py` lifespan 在配置齐全时自动启用）。
**当前唯一阻塞：尚未取得心跳凭据**，导致 staging 心跳从未真正运行过。

---

## 二、mfg 侧现状（自查，供受理方判断）

| 项 | 状态 | 证据 |
|---|---|---|
| 心跳客户端 SDK | ✅ 已实现，已接 lifespan | `backend/heartbeat_client/heartbeat_client.py`；`main.py:33-49` |
| 上报周期/重试/退避 | ✅ 已按契约 1h / 3 次 / 指数退避 | `HEARTBEAT_INTERVAL_SECONDS=3600` 等 |
| 本地 License 字段 | ✅ 已有（读数） | `tenants.license_status`、`tenants.license_expires_at` |
| **staging 心跳运行** | ❌ **未启用** | 容器 env `HEARTBEAT_API_KEY`/`DEPLOYMENT_ID` 为空，日志 `[INFO] 心跳上报未启用（缺少...）` |
| **回传落库** | ❌ 未接（我方待办 P0-2） | `send_once` 取 `resp.json()` 后直接 return；本地字段零写入点 |

---

## 三、申请清单

### A. 心跳凭据（需发放）

| # | deployment_id（建议） | 环境 | tenant_id | product | version | 用途 |
|---|---|---|---|---|---|---|
| A1 | `mfg-staging-01` | staging（mfg1.ziwi.cn，CVM） | `mfg_stage` | `mfg` | 当前版本 | 打通链路、验证上报与回传 |
| A2 | `mfg-prod-01` | 生产（mfg.ziwi.cn，尚未部署，预留） | `mfg_demo` | `mfg` | 当前版本 | 上线时启用 |

每项需要：
1. `HEARTBEAT_API_KEY`（契约 §D.1 称 `license_key`，用作请求头 `X-Api-Key`）
2. `HEARTBEAT_DEPLOYMENT_ID`（与上表一致）
3. 服务端已注册该 deployment（或允许首次上报 auto-seed）

### B. License 记录（需在 heartbeat 后台建档）

| # | tenant_id | product | 申请 status | 申请 expires_at | 说明 |
|---|---|---|---|---|---|
| B1 | `mfg_stage` | `mfg` | `trial`（或 `active`） | 建议给一个**明确的将来日期**（如 `2027-01-01`） | staging 演示 + 验证到期提醒链路 |
| B2 | `mfg_demo` | `mfg` | `trial`（或按商务定） | 同上 | 生产预留 |

> 说明：契约规定服务端对新部署 auto-seed 为 `none`。若只 seed 不发 License，staging 一上报就是"未授权"，
> 与"演示环境"目标冲突——故**显式申请 B1/B2 两条 License 记录**，而不是依赖 auto-seed。

### C. Token 侧（无需发放，仅需确认）

| # | 事项 | 需确认内容 |
|---|---|---|
| C1 | cloud JWT `products[]` 含 `mfg` | mfg 走 cloud 登录的用户，其 `users.products` 是否已含 `mfg`；如未含，申请登记 |
| C2 | JWKS | 当前 `kid` 是否为 `key_v1`；`GET https://cloud.ziwi.cn/api/v1/auth/public-key` ✅ 可用；issuer 取值 |
| C3 | 有效期 | access_token 1h / refresh_token 7d（RFC 9700 旋转）是否与现状一致 |

---

## 四、需受理方确认的三个语义问题（阻塞我方实施）

> 这三条不确认，我方即使拿到 key 也只能"上报"，无法正确落地本地门禁。

**Q1. License 状态枚举映射**
服务端 `none | trial | active | expired | revoked` ↔ mfg 本地 `null(未配置) | valid | expired | invalid`。
`trial`、`revoked` 在 mfg 侧无对应态。请确认映射表，或允许 mfg 扩充本地枚举（倾向后者：`null | trial | valid | expired | revoked`）。

**Q2. `none` 是否触发降级**
契约 §D 的降级判据是"连续 24h 失联"，而非"License 状态为 none"。
请确认：**License 状态本身（尤其 `none`/`expired`/`revoked`）是否会直接触发 mfg 侧降级限制**？
若会，staging 必须先拿到 B1 的 License 才能开心跳，否则一开就可能被限。

**Q3. 私有部署离线 License 文件格式**
契约 §D.1 要求**首次上报必带** `license_issued_at` / `license_expires_at`，且 §C 提到"离线 License（内嵌 cloud 公钥）"。
请确认：私有部署场景下，这两个值的**权威来源**是什么？（离线 License 文件？后台发放后手工填入 env？）
——目前 mfg 侧这两个 env 为空，若无人填写，新部署会持续报 `needs license info` 而无法注册。

---

## 五、我方承诺（拿到 A/B 后实施）

1. 配置 staging env 并重启 → 验证日志出现「心跳上报客户端已启动」+ 首次上报成功
2. **接通回传落库（P0-2）**：把响应中的 `license_status` / `expires_at` / `revoked` 写回 `tenants.license_status`、`tenants.license_expires_at`，并补 `last_heartbeat_at`
3. 本地门禁按 Q1 确认的映射实施（契约"读到即判"）
4. 全程只在 staging 验证，不动生产；不触碰 `mfg1-db`（只加 env，不改数据卷）

---

## 六、验收标准

- [ ] heartbeat 后台 `/admin/deployments` 出现 `mfg-staging-01`，状态 `online`，有 `last_heartbeat_at`
- [ ] `/admin/licenses` 可见 `mfg_stage / mfg` 记录，status 与 expires_at 与申请一致
- [ ] mfg staging 容器日志出现心跳上报成功（非"未启用"）
- [ ] mfg 本地 `tenants.license_status` / `license_expires_at` 被回写为非空（P0-2 完成后）
- [ ] `/admin/alerts` 无 mfg 相关 critical 告警

---

## 七、回执位（请 codebuddy 填写）

| 项 | 发放值 / 结论 | 时间 |
|---|---|---|
| A1 `mfg-staging-01` API Key | | |
| A2 `mfg-prod-01` API Key | | |
| B1 / B2 License 记录 | | |
| C1/C2/C3 确认 | | |
| Q1 枚举映射 | | |
| Q2 none 是否降级 | | |
| Q3 离线 License 来源 | | |
