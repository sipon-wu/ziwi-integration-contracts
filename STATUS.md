# 跨项目状态看板（STATUS.md）

> 各项目/移交物的实时状态在此显形，避免"纸绿"。格式：`项目 | 状态 | 归属 | 最近进展 | 待办`。

| 项目 | 状态 | 归属 | 最近进展 | 待办 |
|------|------|------|---------|------|
| mfg (WMS) | 已移交·自闭环就绪 | workbuddy | 2026-07-12 方案 A 完成：workbuddy 获 CVM root SSH key + `deploy.sh` 推 github `c21bc65`；预发布已对齐 `origin/main 336899f`（含 N3 修复）、`mfg1-backend` healthy、DB 未动 | workbuddy 自验 key 连通性 + `./deploy.sh` 跑通；闭环 N5 过账/流水（真 token + 对账探针）；修探针路径 `/wms/*`→`/api/v1/wms/*`；过 GATE 后置 `Released` |
| cloud / license 线 | 归属变更·方案已对齐 + **代码仓独立化** | **codebuddy**（2026-07-27 用户拍板，自 workbuddy 移回） | 2026-07-27 完成三方对齐（详见下方决策记录）：school 侧两篇方案修订至 v0.6/v1.1，mfg 仓 v0.2 草案标记过时；**mfg 接入 cloud 契约新增 §I「License 查询与同步接口（预留·Phase 2）」**（`GET /api/v1/tenants/{tenant_id}/licenses` + 本地字段 + 心跳同步机制）。**2026-08-12 追加**：cloud 后端建独立 GitHub 仓 `sipon-wu/ziwi_cloud`（原游离态 CVM `/opt/cloud-idp/backend` 已接本地 git + 备份 + 推送）；**续写脉络**（不另起山头）：归属变更与 2026-08 进展见 `contracts/mfg接入cloud接口契约.md` 头部「⚠️ 归属变更提示」+ §D.4 心跳服务端实现；git 工作流（方案 A）落到 `runbooks/CVM部署通用规范与坑清单.md` G3 坑下 | cloud License 服务/DB（Phase 2）建设；§I 接口落地实现；License 同步通道（webhook vs 心跳）待拍板 |

## 今日服务器策略沉淀（已推 github `a3959f8`，workbuddy 须 pull）
- 新建 `runbooks/CVM部署通用规范与坑清单.md`：权限模型 / 部署副本分离 / 只读 github key / 容器名冲突 / 探针路径纪律 / 完成门槛 DoD
- 更新 `school双环境部署工作流.md`：补 rsync 排除误伤 `cmd/server` 坑（G5）
- 更新 `mfg_移交物_预发布对齐与待办_20260712.md`：标注方案 A 完成
- **codebuddy 侧不再代执行 mfg 部署**，后续全归 workbuddy（注：仅指 WMS 业务线；cloud/license 线 2026-07-27 已由用户拍板移回 codebuddy，见下）

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
