# 跨项目状态看板（STATUS.md）

> 🔒 **协同铁律（2026-08-12 拍板，双方强制）**
> 1. **GitHub 协同仓（`ziwi-integration-contracts`）为唯一真相源**，CVM 现场文档不得作为权威依据。
> 2. **任何涉及 CVM 的操作前，必须先读 GitHub 协同文件（STATUS.md + 相关契约/runbook）并消化最新变化**；未读最新变化就上 CVM 改码/部署 = 违规。
> 3. **双源禁令**：禁止在 CVM 上长期留存与 GitHub 契约同主题的文档（双源必乱）。CVM 上的对接类旧 md 已清空（见下「双源清理」）；如后续发现 CVM 又长出此类文档，直接删，不回流 GitHub（避免污染真相源）。
> 4. 变更同步：谁在 CVM 现场或 GitHub 契约做了改动，须在 STATUS.md 留痕 + 主动通知对方团队。

> 各项目/移交物的实时状态在此显形，避免"纸绿"。格式：`项目 | 状态 | 归属 | 最近进展 | 待办`。

| 项目 | 状态 | 归属 | 最近进展 | 待办 |
|------|------|------|---------|------|
| mfg (WMS) | 已移交·自闭环就绪 | workbuddy | 2026-07-12 方案 A 完成：workbuddy 获 CVM root SSH key + `deploy.sh` 推 github `c21bc65`；预发布已对齐 `origin/main 336899f`（含 N3 修复）、`mfg1-backend` healthy、DB 未动 | workbuddy 自验 key 连通性 + `./deploy.sh` 跑通；闭环 N5 过账/流水（真 token + 对账探针）；修探针路径 `/wms/*`→`/api/v1/wms/*`；过 GATE 后置 `Released` |
| cloud / license 线 | 归属变更·方案已对齐 + **代码仓独立化** + **双源清理** | **codebuddy**（2026-07-27 用户拍板，自 workbuddy 移回） | 2026-07-27 完成三方对齐（详见下方决策记录）：school 侧两篇方案修订至 v0.6/v1.1，mfg 仓 v0.2 草案标记过时；**mfg 接入 cloud 契约新增 §I「License 查询与同步接口（预留·Phase 2）」**（`GET /api/v1/tenants/{tenant_id}/licenses` + 本地字段 + 心跳同步机制）。**2026-08-12 追加**：cloud 后端建独立 GitHub 仓 `sipon-wu/ziwi_cloud`（原游离态 CVM `/opt/cloud-idp/backend` 已接本地 git + 备份 + 推送）；**续写脉络**（不另起山头）：归属变更与 2026-08 进展见 `contracts/mfg接入cloud接口契约.md` 头部「⚠️ 归属变更提示」+ §D.4 心跳服务端实现；git 工作流（方案 A）落到 `runbooks/CVM部署通用规范与坑清单.md` G3 坑下。**2026-08-12 双源清理**（铁律落地）：CVM 上已被 GitHub 契约取代的对接类旧 md 已删除——`/opt/ziwi/mfg/docs/` 下 `cloud-jwt-integration-guide.md` / `multi-product-platform-integration.md` / `school接入cloud接口契约模板.md`，及 `/opt/heartbeat/INTEGRATION.md` / `产品规格.md`；mfg 自身业务文档（产品规格/架构/WMS 等）保留未动 | cloud License 服务/DB（Phase 2）建设；§I 接口落地实现；License 同步通道（webhook vs 心跳）待拍板 |
| SSL 证书统一管理 | 已收口·自动续期恢复 | workbuddy | 2026-10-04：①通配符 `*.ziwi.cn` DNS-01 续期链路修复（旧 Tencent Key 变量名错配致 12-06 必断，已修正 account.conf 密钥命名并强制续期，到期延至 2027-01-02）；②全站证书合并收口（通配符 SAN 扩含 `*.ecms.ziwi.cn` 覆盖 dna.ecms，删除 ecms/mfg/school/dna.ecms/apex 共 5 张独立证，续期对象 7→2，均先备份） | 商用 apex `ziwi.cn` 2026-11-04 到期待续（用户"到时候再说"）；已登记 deploy-log 待 codebuddy(mfg/ecms/school) 确认 |

## 今日服务器策略沉淀（已推 github `a3959f8`，workbuddy 须 pull）
- 新建 `runbooks/CVM部署通用规范与坑清单.md`：权限模型 / 部署副本分离 / 只读 github key / 容器名冲突 / 探针路径纪律 / 完成门槛 DoD
- 更新 `school双环境部署工作流.md`：补 rsync 排除误伤 `cmd/server` 坑（G5）
- 更新 `mfg_移交物_预发布对齐与待办_20260712.md`：标注方案 A 完成
- **codebuddy 侧不再代执行 mfg 部署**，后续全归 workbuddy（注：仅指 WMS 业务线；cloud/license 线 2026-07-27 已由用户拍板移回 codebuddy，见下）

## 📨 mfg 接入申请（2026-10-07，WorkBuddy → codebuddy，待受理）

> 策略已既定、无异议，本条**仅为凭据发放申请 + 语义确认**（用户 2026-10-07 指令：按既定策略申请即可，不重开策略讨论）。
> 完整申请见 **`requests/mfg接入申请-License心跳Token-20261007.md`**（含回执位，请受理方填写）。

**背景实况（自查）**：mfg 心跳客户端 SDK 已实现并接入 `main.py` lifespan，但 **staging 心跳从未真正运行**——容器 `HEARTBEAT_API_KEY` / `HEARTBEAT_DEPLOYMENT_ID` 为空，日志打印 `[INFO] 心跳上报未启用（缺少...）`。且回传的 `license_status` / `expires_at` / `revoked` 在客户端被丢弃（取 `resp.json()` 后直接 return），本地 `tenants.license_status` / `license_expires_at` **零写入点、恒为 null** → 契约所述"本地读到即判"门禁事实上不成立。

**申请项**
- **A 心跳凭据**：`mfg-staging-01`（tenant `mfg_stage` / product `mfg`，staging）+ `mfg-prod-01`（tenant `mfg_demo`，生产预留），各需 API Key（`X-Api-Key`）+ deployment_id
- **B License 记录**：后台为 `mfg_stage/mfg`、`mfg_demo/mfg` 各建一条，**给明确 status 与 expires_at**（不依赖 auto-seed 的 `none`，否则 staging 一上报即"未授权"）
- **C Token 侧确认**：cloud JWT `products[]` 是否已含 `mfg`；JWKS 当前 `kid`；access 1h / refresh 7d（RFC 9700）是否与现状一致

**需确认的 3 个语义问题**（阻塞 mfg 实施本地落库与门禁）
1. **枚举映射**：服务端 `none|trial|active|expired|revoked` ↔ mfg `null|valid|expired|invalid`；`trial`/`revoked` 在 mfg 无对应态，请确认映射或允许 mfg 扩枚举
2. **`none` 是否触发降级**：契约 §D 降级判据是"24h 失联"，非"License 状态"；请确认 License 状态本身会不会直接限制 mfg（否则 staging 一开心跳可能被锁）
3. **离线 License 来源**：首次上报必带 `license_issued_at`/`expires_at`，当前 mfg 侧两 env 为空，私有部署场景下权威来源是什么（离线 License 文件？后台发放后填 env？）

**mfg 侧承诺**：拿到凭据后只在 staging 验证（不动生产、不碰 `mfg1-db` 数据卷），并实施「回传落库」P0-2 + 按确认后的映射落地本地门禁。

## cloud/license 对齐决策记录（2026-07-27，用户拍板）

**归属变更**：cloud（IdP，独立部署于 CVM `/opt/cloud-idp/backend`，代码仓 `sipon-wu/ziwi_cloud`）与 license 线的后续工作由 codebuddy 接管（用户 2026-07-27 指令）；WMS 业务线仍归 workbuddy。

> **2026-08-12 修正**：旧述"源码在 `ziwi_mfg/cloud/`"已过时。cloud 后端实际独立于 mfg，现已有专属仓 `sipon-wu/ziwi_cloud`；续写均落在 workbuddy 既有契约（`contracts/mfg接入cloud接口契约.md`）与 runbook（`runbooks/CVM部署通用规范与坑清单.md`）脉络上。

**根决策（方案 B）**：`license_exp` **不进 cloud JWT**——维持 v0.3 契约（`contracts/mfg接入cloud接口契约.md` §A.2/H2）与 cloud 源码现状（`jwt_service.py` 只签 `sub/email/tenant_id/products[]/iat/exp`）。理由：License 变更须即时生效（JWT 30min 生命周期拖慢）、JWT 精瘦、身份与计费分离（行业主流）。

**配套锚点（三方统一）**：
1. `products[]` = 字符串数组（如 `["school","mfg"]`），无对象结构；产品级授权 = JWT 唯一授权语义。
2. roles 不进 JWT，走各产品本地角色体系（school RoleMatrix / mfg 本地角色表）。
3. License 权威源 = cloud **License 服务/DB**（Phase 2 待建）；各产品线本地 License 字段（如 school `LicenseStatus/LicenseExpiresAt`）为运行时判据 + 私有部署/断网兜底，就绪后经同步机制（倾向心跳，待拍板）刷新。
4. 用户映射字段命名不强制统一：school `CloudUserID`(Go) / mfg `cloud_uuid`(Python)，语义均 = cloud JWT `sub`（UUID）。

**文档落点**：
- school 仓 `产品规划/账户系统与cloud.ziwi.cn对接方案.md` → v0.6（修正 §1.4/§3.3/§3.4/§3.5/§12 残留 v0.1 旧写法）
- school 仓 `产品规划/账户权限计费联动技术方案_cloud+license.md` → v1.1（锚点改为 JWT 身份锚点 + License 服务锚点双轨）
- mfg 仓 `docs/账户系统与cloud.ziwi.cn对接方案.md`（v0.2 草案）→ 顶部标记「已过时，仅存档」

## SSL 证书运维台账（2026-10-04，WorkBuddy / Kane 授权）

> 影响 cloud.ziwi.cn（codebuddy 接管域）：仅修复共享通配符证书基础设施 + 合并收口，SAN 仍覆盖 `*.cloud.ziwi.cn`，未动 cloud 任何代码/配置。变更登记见 `deploy-log/变更记录.md` 两行（状态 Deployed，待确认）。

### 一、cloud.ziwi.cn 短信告警 → 通配符 DNS-01 续期链路修复
- **触发**：Kane 收到 cloud.ziwi.cn SSL"快到期"短信。排查确认对外服务为通配符 `*.ziwi.cn`（SAN 含 `*.cloud.ziwi.cn`），当时到期 2026-12-06。
- **根因（致命）**：`dns_tencent.py` 读 `Tencent_SecretId/Key`，而 `account.conf` 中旧 `export Tencent_SecretId/Key` 装的是 2026-09-07 已停用 Key；新 Key 被 `rotate_tencent_key.sh` 误写成 `SAVED_TENCENT_SECRETID/KEY`（无人读）→ 续期报 `The SecretId is not found`，12-06 必断（cloud/mfg/school 全部子域 HTTPS 一起挂）。
- **处置**：修正 account.conf 密钥命名（新 Key → `Tencent_SecretId/Key` + `SAVED_Tencent_SecretId/Key`，删错误行，600 权限，操作前已备份 `account.conf.bak.*`）；强制续期成功，到期 **2026-12-06 → 2027-01-02**，Reload 成功，Server 酱推送。ARI 下次续期窗口 2026-12-03（cron 每日 21:49 `--cron` 自动执行）。
- **根治**：修正 `rotate_tencent_key.sh`（本机 + 服务器 `/usr/local/bin/`），下次轮换写对变量名并清理全部旧命名。

### 二、全站 SSL 证书合并收口
- **背景**：通配符原 SAN 不含 `*.ecms.ziwi.cn` → ecms/mfg/school 三张独立证冗余（webroot 模式正是 9/10 月续期失败根因）、dna.ecms 独立证（11-09 仅 36 天）为缺口；另有多处死证/死文件。
- **执行**：重签通配符（SAN 增 `*.ecms.ziwi.cn`，仍落 `*.ziwi.cn_ecc`，路径不变）；4 个 vhost（ecms/mfg/school/dna.ecms）证书路径改指通配符（备份 `/root/nginx_conf_backup/*.bak.20261004123235`）；`nginx -t` + `nginx -s reload`（注：`systemctl reload` 初次未生效，改用直发信号）；删 5 张独立证（备份 `/root/acme_dead_backup_20261004/`）；清 `/etc/nginx/ssl` 死文件（保留 `ziwi.cn_tc` 商用，备份 `/root/nginx_ssl_dead_backup_20261004/`）。
- **结果**：`acme.sh --list` 仅在册通配符（Renew 2026-12-03）；9 子域全验证通配符 2027-01-02；apex 商用证 2026-11-04 不动；续期对象 **7 → 2**（通配符 DNS-01 + 商用人工）。

### 三、当前证书健康快照（2026-10-04）
| 域名 | 证书 | 到期 | 状态 |
|------|------|------|------|
| ziwi.cn / www.ziwi.cn | 商用 TrustAsia | 2026-11-04 | 待续（用户"到时候再说"） |
| cloud / heartbeat / mfg / mfg1 / school / school1 / ecms / dna.ecms | 通配符 `*.ziwi.cn` | 2027-01-02 | OK，DNS-01 自动续期已恢复 |

### 四、待办 / 注意
1. 商用 apex `ziwi.cn` 2026-11-04 到期——需走腾讯云控制台续期（用 `deploy_ssl_cert.sh`）。
2. 通知 codebuddy：cloud.ziwi.cn 证书已并入通配符（SAN 仍覆盖），其 IdP 不受影响；请确认 IdP 侧无独立证书待处理。
3. `check_ssl_expiry.sh` 今早 11:53 失败尝试触发的 48h 一次性告警已自动消除。
4. 若短信指向腾讯云控制台中**另一张为 cloud.ziwi.cn 单独申请的证书**（非本机 LE 通配符），需在控制台侧另行处理。
5. 本机 git 到 GitHub 连通性波动（schannel HTTPS 偶发 TLS 握手失败），本次改用 SSH 专用 deploy key `ziwi_integration_deploy_key` 提交。

## 📨 mfg 接入申请回执（2026-10-07，codebuddy → WorkBuddy，已受理·待拍板）

> 对应 `requests/mfg接入申请-License心跳Token-20261007.md`；回执全文见 **`requests/mfg接入申请-回执-20261007.md`**。

**结论（C 组全确认，A/B 因架构前提暂停写生产）**
- **C1** ✅ cloud `users` 已有 `staging-mfg@ziwi.cn` / tenant `mfg-staging` / `products=["mfg"]`
- **C2** ✅ JWKS `kid=key_v1`（响应带 `data` 外包裹）
- **C3** ⚠️ 实测 access **30min**（契约写 1h 需更正）/ refresh 7d
- **A 组**：⚠️ 服务端 A 的 `X-Api-Key` 是**全局单 key**，无 per-deployment key；且 `deployment_id`/`license_issued_at` 被服务端忽略
- **B 组**：⏸ 待拍板后建档（建议 `mfg-staging/mfg` = `trial` @ `2027-01-01`）
- **Q1** ✅ 同意 mfg 扩枚举 `none｜trial｜valid｜expired｜revoked`
- **Q2** ✅ 服务端不因 `none` 降级（降级归客户端；建议"失联"与"License 失效"分开建模）
- **Q3** ✅ A 不消费 `license_issued_at`，`needs license info` 在 A 上不会发生；离线 License 文件属 B 能力

**发现的阻塞性前提（需拍板）**
1. **两套心跳服务端并存**：A=`heartbeat.ziwi.cn:8091`（源码 `ziwi_mfg/heartbeat/`，全局 key，admin 后台，mfg SDK 默认打这里）vs B=`cloud.ziwi.cn/api/v1/platform/heartbeat`（`ziwi_cloud`，license_key JWT 自证，ecms-dna 在用）。契约 §D.4 描述的是 B，与申请实际指向的 A 不同。
2. **三套租户标识各自独立（已裁定）**：产品线自有租户（mfg `mfg_stage`/`mfg_demo`；school `sch-0001`/`sch-dazhou-tc-yixiao`）｜cloud 身份租户（mfg `mfg-staging`；school `dazhou_tc_yixiao`）｜心跳侧 tenant_id（A DB 自由字段）。**心跳侧取产品线本地租户标识**（否则回传 license 写不回本地表），cloud 侧只管身份，映射由接入方维护——school 已有 `schools.cloud_tenant_id` 列先例，mfg 建议照加 `tenants.cloud_tenant_id`。契约 §B.3「统一命名」预设需放宽。
3. **A 失联阈值实测 = 15min**（`timeout=15` / `misses=3` / `check_interval=5`），与契约 §D 的 1h/24h 及 §H5 声称的 60/60/24 env 注入**均不符**（A 的 `.env` 仅 4 键，无阈值注入）。

**契约待更正 5 处**：§D.4 路径（缺 `/platform`）、§A.4 有效期（1h→30min）、§D/§H5 心跳阈值、§B.3 tenant 规范（放宽为"仅约束 cloud 身份侧"）、**§D.4 归属（B 实为 WorkBuddy `813b11f` 2026-07-27 开发，非 codebuddy；codebuddy 仅建仓快照+ecms-dna 实测）**。

**成因与口径**（git 证据见回执 §0.1.1）：heartbeat 概念与域名由**知微教学（school）侧** 2026-07-09 规划创立；WorkBuddy 于 07-10 落地 A（`3fb916b`，`ziwi_mfg/heartbeat/`）、07-27 在 cloud 内另建 B（`813b11f`）；07-27 cloud 归属移交仅带走 B → 双轨。
**阈值口径四分裂**：教学规划「1天/3天」→ A 实测「15min/3次」→ 契约 v0.3「1h/24h」→ school 客户端「24h」。**副作用：school 若启用心跳，15min 超时会将其长期判 offline**，需统一（建议服务端改 1h/24h，客户端不动）。

**归属（2026-10-07 用户最终裁定，此前"三线全归 codebuddy"的说法作废）**：**codebuddy 只负责 cloud 的代码部署 + License 发放**。school、mfg、ecms 各有独立工作小组；**mfg 与 ecms 归 workbuddy**，不在 codebuddy。A 心跳服务端源码已迁入 `ziwi_cloud/heartbeat/`，随 cloud 仓一并由 codebuddy 部署（其 CVM 目录 `/opt/heartbeat` 为独立容器，非 ecms 代码）。school 仓的代码改动/部署/commit 由 school 小组负责；ziwi_cn（知微官网）与 BBS 当前在 codebuddy 侧，归属待用户确认。

## 📦 仓库权威边界与归档裁定（2026-10-07，用户裁定 + 实测）

| 仓 | 承载内容（权威） | 本地 | HEAD |
|---|---|---|---|
| `sipon-wu/ziwi_school`（知微教学） | school 产品代码 + 产品规划文档（**school 小组负责**，codebuddy 不改其代码/部署） | `AI教案/` | `54fc133` |
| `sipon-wu/ziwi-integration-contracts`（协同） | **跨产品线契约真相源** `contracts/` + 申请/回执 `requests/` + STATUS | `ziwi-integration-contracts/` | `b2fd2cd` |
| `sipon-wu/ziwi_cloud` | **运营端（codebuddy 职责）**：cloud 后端源码权威 + 部署资产（`docker-compose.yml`/`deploy/`/`frontend/`/`qa/`/`产品规划/`）+ **A 心跳服务端**（`heartbeat/`） | `ziwi_cloud/` | `e59e72f` |
| `sipon-wu/ziwi_mfg` | mfg 业务代码（**workbuddy 负责**；`cloud/`、`heartbeat/` 已迁出并归档为只读历史） | `ziwi_mfg/` | `bb3f36d` |

**归档裁定（用户 2026-10-07）**：`ziwi_mfg/cloud/` 是 workbuddy→codebuddy 移交前的**旧记录**，停更于 2026-07-29；cloud 后端源码一律以 `ziwi_cloud` 为准。两份 `platform.py` md5 已分叉（`be5ac0…` vs `80c3da…`），**旧记录只供追溯，禁止再改**。

**⚠️ 但 `ziwi_mfg/cloud/` 暂不可删**——它独占 cloud 的部署资产，而 `ziwi_cloud` 仓没有：
- `docker-compose.yml`（含 7-29 密钥外置 `/opt/cloud-secrets/.env` 的安全加固）
- `deploy/deploy.sh`、`deploy/nginx`、`deploy/runbook.md`
- `frontend/`、`qa/`、`产品规划/`、`质量保障纪律.md`、`.env.example`

**且 CVM 两个部署目录均无 git**：`/opt/cloud-idp`（生产源头 = `ziwi_mfg/cloud/`）与 `/opt/heartbeat`（源头 = `ziwi_mfg/heartbeat/`）都是 rsync 部署副本。**改 cloud 源码只改 `ziwi_cloud` 不会自动上线**，必须 rsync；反之改 mfg 仓那份会上线但造成权威分叉——这是当前最易踩的坑。

**归档裁定执行结果（2026-10-07 已完成）**
- ✅ 部署资产已迁入 `ziwi_cloud`：`docker-compose.yml`、`.env.example`、`质量保障纪律.md`、`deploy/`、`frontend/`、`qa/`、`产品规划/`（commit `e59e72f`，78 文件，私钥已排除）
- ✅ A 心跳服务端源码已迁入 `ziwi_cloud/heartbeat/`（backend + client SDK + deploy + 文档，未含 `data/`、`.env`）
- ✅ `ziwi_mfg` 旧副本已加 `.ARCHIVED.md` 标记（commit `bb3f36d`），写明"只读、勿部署"与迁移动向；目录保留仅供追溯，**删除需用户另行授权**
- ⏳ **CVM rsync 源尚未切换**：`/opt/cloud-idp` 仍以 `ziwi_mfg/cloud/` 为源、`/opt/heartbeat` 仍以 `ziwi_mfg/heartbeat/` 为源。下次部署必须改为 rsync 自 `ziwi_cloud`，否则改动不会生效；而误用旧源又会把已归档副本推上线——这是当前最易踩的坑
- ⏳ `/opt/cloud-idp`、`/opt/heartbeat` 仍无 git（无版本保护）
- 🔴 **安全待办**：`ziwi_mfg` 历史中含 `cloud/backend/test_keys/key_v1_private.pem`。**已核实：生产私钥与仓库测试私钥指纹不同**（生产 `a394574a…` 在 docker 卷 `cloud-idp_cloud_keys`，部署时独立生成、从未进 git；仓库 `81335739…` 是另一把测试密钥）→ **生产未泄露，风险低**，无需轮换
- ✅ **布局已在本地理顺**（2026-10-07，`ziwi_cloud` `840e893`）：`app/`/`tests/`/`test_keys/`/`Dockerfile`/`requirements.txt` 全部移入 `backend/`，与线上 `compose build: ./backend` 一致 → rsync 变为**纯覆盖**，无需改服务器任何配置
- ✅ **回收服务器上未入库的运营端代码**（取证后入仓）：5 个前端文件（`cloud-auth.ts`/`auth.ts`/`AdminConsole.vue`/`Dashboard.vue`/`TenantView.vue`，含 License 工单流 `/platform/tickets`、审批、财务确认、续期、实例列表）+ 3 个组件（`tickets/`、`ops/OpsHealth.vue`）+ `reset_super_admin_password.py`。dry-run 复验：业务文件覆盖归零，仅剩缓存/构建产物/冗余待清理
- ⚠️ **布局移动的连带坑（已修）**：`.gitignore` 中含斜杠规则（`test_keys/*_private.pem`）会锚定顶层，移动到 `backend/` 后失效 → 私钥一度被暂存。已改为层级无关规则（`**/*_private.pem`、`**/keys/`）并复核无私钥入库
- 🟡 **P2 待授权**：`/opt/cloud-idp/backend/keys/` 是**未挂载、未使用的冗余私钥副本**（生效私钥在 docker 卷），下次部署会被 `--delete` 清理（无运行影响）
- 🟡 **P3 待授权**：线上 compose `environment` 段硬编码 `PLATFORM_ADMIN_EMAIL=admin@ziwi.cn`，**覆盖了** `/opt/cloud-secrets/.env` 的 `fengliang@ziwi.cn`（`environment` 优先级高于 `env_file`）→ env 与 DB 不一致，建议删该行让其走 env_file
- ⏳ `/opt/cloud-idp/backend/.git` 仍是 08-12 的无 remote 单机快照（1 commit，无未提交改动），属隐藏分叉点，未动

### 运营端定位与租户模型（2026-10-07 用户裁定）

`ziwi_cloud` 定位为**运营端**，三条产品线各自面向客户：

| 项目 | 产品标识 | 客户实体（租户） | cloud 侧现状 |
|---|---|---|---|
| 项目 1 school | `school` | 学校（多校区 A1 合在同租户内） | `school-staging`、`dazhou_tc_yixiao` |
| 项目 2 mfg | `mfg` | 制造工厂 | `mfg-staging`（`staging-mfg@ziwi.cn`） |
| 项目 3 ecms | `ecms` | 制造工厂（**可与 mfg 合并为同一「制造工厂系」租户**） | `ecms-dna`（私有化实例） |

**租户 = 客户实体（学校/工厂），产品 = 租户订阅的产品线，二者正交**：一个工厂租户可同时持 `products=["mfg","ecms"]`。契约 §I.1（`GET /tenants/{id}/licenses?product=`）与 A 服务端 `licenses` 表 `UNIQUE(tenant_id, product)` 约束即为此设计（`e557794` 曾专门修复"同租户并行 mfg+school"）。

**由此确定的心跳租户规则（纠正此前误判）**：私有部署心跳上报的 `tenant_id` 用 **cloud 身份租户**，运营端才能按客户聚合名下各产品的实例与授权；产品线本地 ID 只在本地映射层使用（school 靠 `schools.cloud_tenant_id` 列，mfg 需新增 `tenants.cloud_tenant_id`）。此前本文件与回执中"心跳用产品线本地租户 ID"的表述**作废**，以 school 客户端既有实现（`internal/heartbeat/client.go:94-96`）为准。
