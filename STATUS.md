# 跨项目状态看板（STATUS.md）

> 各项目/移交物的实时状态在此显形，避免"纸绿"。格式：`项目 | 状态 | 归属 | 最近进展 | 待办`。

| 项目 | 状态 | 归属 | 最近进展 | 待办 |
|------|------|------|---------|------|
| mfg (WMS) | 已移交·自闭环就绪 | workbuddy | 2026-07-12 方案 A 完成：workbuddy 获 CVM root SSH key + `deploy.sh` 推 github `c21bc65`；预发布已对齐 `origin/main 336899f`（含 N3 修复）、`mfg1-backend` healthy、DB 未动 | workbuddy 自验 key 连通性 + `./deploy.sh` 跑通；闭环 N5 过账/流水（真 token + 对账探针）；修探针路径 `/wms/*`→`/api/v1/wms/*`；过 GATE 后置 `Released` |

## 今日服务器策略沉淀（已推 github `a3959f8`，workbuddy 须 pull）
- 新建 `runbooks/CVM部署通用规范与坑清单.md`：权限模型 / 部署副本分离 / 只读 github key / 容器名冲突 / 探针路径纪律 / 完成门槛 DoD
- 更新 `school双环境部署工作流.md`：补 rsync 排除误伤 `cmd/server` 坑（G5）
- 更新 `mfg_移交物_预发布对齐与待办_20260712.md`：标注方案 A 完成
- **codebuddy 侧不再代执行 mfg 部署**，后续全归 workbuddy
