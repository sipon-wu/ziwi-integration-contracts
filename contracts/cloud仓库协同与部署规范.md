# cloud.ziwi.cn 代码仓库协同与部署规范

> 版本：v1.0 ｜ 日期：2026-08-12 ｜ 起草：codebuddy
> 关联：`ziwi_cloud` 仓 README.md（代码仓内同源文档）
> 配套：`runbooks/CVM部署通用规范与坑清单.md`（CVM 部署通用纪律）

## 0. 背景

cloud（IdP，对外 `cloud.ziwi.cn`）后端长期以**游离态**运行于 CVM `/opt/cloud-idp/backend`，**无 git 版本保护**。2026-08-11 用户拍板为其建独立 GitHub 仓 `sipon-wu/ziwi_cloud`，并由 codebuddy 完成：①本地备份 ②GitHub 初始化并推送 ③CVM 接本地 git。

> **重要修正**：此前 `STATUS.md` 称 "cloud 源码在 `ziwi_mfg/cloud/`" 已过时。cloud 后端实际**独立部署于 CVM `/opt/cloud-idp/backend`**，现已有独立仓 `ziwi_cloud`，与 mfg 代码分离。

## 1. 四节点关系

| 节点 | 角色 | 说明 |
|------|------|------|
| **本地 Mac** (`/Users/sipon/CodeBuddy/ziwi_cloud/`) | 操作枢纽 | 写码、rsync 到 CVM、**唯一持有 GitHub SSH key（可 push GitHub）** |
| **GitHub** (`sipon-wu/ziwi_cloud`) | 代码真相源/备份 | 只存代码不运行；**不含任何私钥** |
| **腾讯云 CVM** (`193.112.163.147`) | 运行现场 | docker 容器实际运行处；含 DB、JWT 签名密钥卷、`keys/` RSA 私钥 |
| **workbuddy（Win）** | 协同开发 | 共用同一 CVM 与 GitHub 账号；**但无 CVM SSH key、无 GitHub push 能力** |

**流转模型**：Mac 写 → GitHub 存 → CVM 跑。GitHub ≠ 生产，代码须落到 CVM 并 `docker compose up --build` 才生效。

## 2. Git 工作流（方案 A：CVM 不直连 GitHub）

> CVM 只建**本地 git**（版本保护）；所有 GitHub 同步经 Mac 枢纽。这是刻意的安全设计——不给 CVM 配 GitHub key，否则 workbuddy 可借同一 CVM 拿到 push 你所有仓库的能力。

### 2.1 CVM 上改码（本地提交）
```bash
cd /opt/cloud-idp/backend
# 改码后 —— 严禁 git add -A / git add .，只加具体路径
git add <具体文件或目录>
git commit -m "feat: ..."
```

### 2.2 同步到 GitHub（在 Mac 上执行）
```bash
# 1. 从 CVM 拉回最新（已排除 venv/私钥/本地库）
rsync -az --delete \
  --exclude='.venv' --exclude='__pycache__' --exclude='*.pyc' \
  --exclude='keys' --exclude='cloud_local.db' --exclude='.pytest_cache' \
  root@193.112.163.147:/opt/cloud-idp/backend/ /Users/sipon/CodeBuddy/ziwi_cloud/

# 2. Mac 上提交并推送
cd /Users/sipon/CodeBuddy/ziwi_cloud
git add <具体路径>
git commit -m "..."
git push origin main
```

## 3. 安全纪律（红线）

1. **永不 `git add -A` / `git add .`** —— 必须列具体路径，防 `keys/` 私钥入历史。
2. **`.gitignore` 已挡**：`keys/`、`test_keys/*_private.pem`、`.venv/`、`cloud_local.db`、`*.db`、`.env`。
   - 仅 `test_keys/key_v1_public.pem`（**公钥，无害**）入库。
3. **私钥只活 CVM 本地**（`/opt/cloud-idp/backend/keys/key_v1_private.pem`），不经 Mac、不进 GitHub。
4. **CVM 无 GitHub SSH key**（PB denied），不要在 CVM 上 `git push`（会失败）；需要进 GitHub 由 Mac 枢纽负责。
5. JWT 签名密钥在 `cloud_keys` docker 卷；DB 口令在 `/opt/cloud-secrets/.env`（均 CVM 本地，不在本仓）。

## 4. workbuddy 团队须知

- 可在 **CVM 上直接改码并 `git commit`（本地版本保护）**，但不要试图在 CVM 上 `git push`（无 key，失败）。
- 需进 GitHub 时：把改动留在 CVM 本地 commit，由 **Mac/codebuddy 负责 rsync + push**（唯一有 GitHub key 的枢纽）。
- 任何涉及 `keys/`、`cloud_local.db`、`.env` 的操作都不要 commit。
- 部署生效（CVM）：`cd /opt/cloud-idp && docker compose up -d --build backend`（参照 `runbooks/CVM部署通用规范与坑清单.md`）。

## 5. 仓库结构（速览）

```
app/{api,core,models,schemas,services}/   # FastAPI 路由/核心/JWT/RSA/模型/业务
app/main.py  app/config.py  app/seed_platform.py
tests/  test_keys/(仅公钥)  Dockerfile  requirements.txt  .gitignore
```
私有化实例心跳：`POST /api/v1/heartbeat`（实例用 license_key 自证）+ `GET /api/v1/instances/heartbeats`（平台角色查看在线）。
