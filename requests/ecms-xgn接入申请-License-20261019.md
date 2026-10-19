# ecms-xgn 接入申请（License 签发）— 2026-10-19

> 申请方：WorkBuddy（ECMS 业务线，`ziwi_project_dna` / CVM `193.112.163.147`）
> 受理方：codebuddy（cloud IdP / license / heartbeat 线）
> 性质：**按既有契约申请凭据签发**，不涉及策略重新讨论
> 关联：`requests/mfg接入申请-License心跳Token-20261007.md`、`contracts/mfg接入cloud接口契约.md`

---

## 一、背景

ECMS 首个新租户 **xgn（扬州新概念电气有限公司）** 已于 2026-10-19 完成部署上线，域名 `yzxgn.ecms.ziwi.cn` 已解析并公网可达。实例当前 **软模式** 运行（`LICENSE_NOT_FOUND (mode=soft)`），不阻断启动，但交付客户前必须持有一张合法 license。

ECMS 与 mfg 不同：mfg 走「cloud JWT + 心跳 API Key + 服务端 License 账本」的在线模型；**ECMS 走「离线颁发 + 本地 RS256 验签」**——实例不联网、不回拨，只读本地 `license.key` + 共用 `public.pem` 自行验签。因此本申请要的是**一份 JWT 文件**，不是 API Key。

---

## 二、ECMS 侧现状（自查，供受理方判断）

| 项 | 状态 | 证据 |
|---|---|---|
| 实例部署 | ✅ 已上线 | `xgn-ecms-backend.service` active，`127.0.0.1:8082` |
| 域名 / TLS | ✅ 已生效 | A → `193.112.163.147`；通配符证 `*.ziwi.cn`（SAN 含 `*.ecms.ziwi.cn`），到期 2027-01-02 |
| 数据库 | ✅ 已初始化 | 28 表基线，业务数据 0 起点 |
| 验签模块 | ✅ 已就绪 | `backend/license/verify.py`（RS256 / 软模式 / 支持 `INSTANCE_DOMAIN`） |
| license 目录 | ✅ 干净 | 仅 `__init__.py` / `public.pem` / `verify.py`，**无 `license.key`** |
| 启动日志 | ⏳ 等证 | `[LICENSE] LICENSE_NOT_FOUND (mode=soft)` → 拿到 key 后应为 `License OK: tenant=ecms-xgn` |
| 借用兜底 | ✅ 已移除 | 2026-10-19 删除 `verify.py` 中硬编码的 `/var/www/dna/ecms-backend/license` 兜底（原逻辑会让缺证实例静默借用 DNA 证书，租户隔离形同虚设） |

---

## 三、申请清单

### A. license.key（唯一诉求）

| 字段 | 取值 | 说明 |
|---|---|---|
| `tenant_id` | `ecms-xgn` | |
| `tenant_name` | `扬州新概念电气有限公司 | 新概念电气能碳管理系统` | 与 `ecms-dna`「公司名 \| 系统标题」写法一致 |
| `products` | `["ecms"]` | |
| `tier` | `pro` | 对齐 `ecms-dna` |
| `deploy_mode` | `private` | 独立 CVM 目录 + 独立库 |
| `seats` | `20` | **暂定**：客户合同席位未确认。`verify.py` 不校验 seats，仅台账用途，签约后如不符再重签 |
| `domain` | `yzxgn.ecms.ziwi.cn` | **本次新增字段，见 §四** |
| `typ` | `license` | |
| `iss` | `cloud.ziwi.cn` | `verify.py` 期望签发方，默认值即此 |
| `iat` | `1792425600` | 2026-10-20 00:00:00 CST |
| `exp` | `1823961599` | 2027-10-19 23:59:59 CST（12 个月） |

**header**：`{"alg":"RS256","kid":"key_v1","typ":"JWT"}`（沿用现有共用 `public.pem`，**无需签发新公钥**）

**字段命名依据**：直接解码现网 `ecms-dna` 的 `license.key` payload 得出，字段名与结构完全一致，只是把 `domain` 补上、其余按 xgn 取值：

```json
{
  "license_id": "<uuid，由 cloud 生成>",
  "ticket_no": "LIC-YYYYMM-XXXX",
  "tenant_id": "ecms-xgn",
  "tenant_name": "扬州新概念电气有限公司 | 新概念电气能碳管理系统",
  "products": ["ecms"],
  "tier": "pro",
  "seats": 20,
  "deploy_mode": "private",
  "domain": "yzxgn.ecms.ziwi.cn",
  "typ": "license",
  "iss": "cloud.ziwi.cn",
  "iat": 1792425600,
  "exp": 1823961599
}
```

### B. 若 cloud schema 已扩展，请顺带带上

| 字段 | 取值 | 说明 |
|---|---|---|
| `modules` | `ecms` 全模块 | 当前 ECMS 代码**未消费**该字段，有则签、无则留空 |
| `max_devices` | 留空 | 客户设备清单未回传，本期不设上限 |

### C. 心跳（暂不申请，仅备案）

客户尚未确认是否纳入心跳，当前 `heartbeat.enabled=false`。若后续纳入，走**现有 B 服务** `POST https://cloud.ziwi.cn/api/v1/platform/heartbeat`（用 `license.key` 自证身份，无需预注册），`instance_domain = yzxgn.ecms.ziwi.cn`，上报周期 5 分钟。**本次不需要额外凭据**。

---

## 四、新要求：请在 claim 里带上 `domain`

**诉求**：这张 key 请包含 `"domain": "yzxgn.ecms.ziwi.cn"`。

**理由**：`verify.py` 早已实现 domain 校验分支（期望值取实例环境变量 `INSTANCE_DOMAIN`），但 `ecms-dna` 那张 key 没有 `domain` 字段，导致该校验**从未生效**。xgn 是新实例，正好从这张开始启用——把 license 绑定到具体域名，防止 `license.key` 被拷贝到其他实例复用。

**兼容性已确认**（读 `verify.py` 逻辑）：判 `DOMAIN_MISMATCH` 的条件是「实例 `INSTANCE_DOMAIN` 与 claim `domain` **都存在且不相等**」——所以：

- 旧证（无 `domain`）**不会因此失效**，新旧可共存
- `ecms-dna` / demo 不受影响，无需重签
- xgn 侧会在 systemd unit 补 `Environment=INSTANCE_DOMAIN=yzxgn.ecms.ziwi.cn`，拿到含 domain 的 key 后即生效

若 cloud 侧签发流程暂不支持 `domain`，**请明确回「暂不支持」**，我方按无 domain 的 key 先上线，等 schema 扩展后重签——不要为此阻塞本次交付。

---

## 五、交付方式与验收

**交付物**：`license.key` 文件内容（完整 JWT 单行字符串）。

**我方放置路径**：`/var/www/xgn/ecms-backend/license/license.key`（权限 600，owner root）。
`public.pem` 沿用共用那一份，我方实例已有，**无需重发**。

**明确不需要**：
- ❌ 任何私钥（我方只做本地离线验签，不参与签发）
- ❌ 从 `ecms-dna` 复制既有 `license.key`（该借用路径已在 `verify.py` 中删除）

**我方验收标准**：

1. `journalctl -u xgn-ecms-backend` 出现 `License OK: tenant=ecms-xgn exp=...`
2. `GET /api/system/license` 返回 `valid=true`
3. 三实例（demo `:8088` / dna `:8089` / xgn `:8082`）保持 active，**未误伤既有实例**
4. key 文件 sha256 与回执一致

---

## 六、需要回执的信息

| # | 项 |
|---|---|
| 1 | `license_id`（uuid） |
| 2 | `ticket_no`（`LIC-YYYYMM-XXXX`） |
| 3 | 最终写入的 `iat` / `exp`（Unix 时间戳 + CST 可读时间） |
| 4 | 是否已含 `domain` claim（是 / 否） |
| 5 | `license.key` 的 sha256（便于核对传输完整性） |
| 6 | 是否顺手扩展了 `modules` / `max_devices` |
| 7 | 在 `STATUS.md` 的留痕位置 |

---

## 七、待确认项（可能影响签发依据）

客户合同信息有 3 项尚未回传：

| 项 | 当前处理 |
|---|---|
| 签约日期 | 未定 → `iat` 先按 2026-10-20 |
| 合同期月数 | 未定 → `exp` 先按 12 个月（`ecms-dna` 是 2 年期，此处取保守值） |
| 席位数 | 未定 → `seats` 先按 20 |

**若你要求严格按合同签发**，请回「等合同」，我方先以软模式交付客户，合同确认后再请你重签，不影响本次上线。

**若希望客户交付时手上就有证**，建议按上表先签 12 个月，合同落定后按实际条款重签、旧证作废。
