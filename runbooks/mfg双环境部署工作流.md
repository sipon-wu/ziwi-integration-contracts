# 知微 mfg 制造平台 — mfg1 预发布 / 生产 部署工作流

> 目的：让任何接入 mfg 的团队照此流程部署，**不重复踩已知坑**。
> 配套：school 双环境部署工作流（同机部署参照）、流程闭环规范第三章（部署变更生命周期）。
> 服务器：193.112.163.147（腾讯轻量云），与 school / cloud / heartbeat 共享 host nginx。

---

## 一、方案目的

| 目标 | 说明 |
|------|------|
| **安全发布** | 改动先在 `staging(mfg1)` 验证，通过后再发 prod |
| **镜像隔离** | docker-compose 项目名区分 staging/prod，防止镜像互相覆盖 |
| **数据隔离** | staging 独立数据库，与 prod 互不影响 |
| **证书策略** | 复用通配符 `*.ziwi.cn_ecc`（acme.sh DNSPod DNS-01 签发） |
| **域名分流** | host nginx 按 `server_name` 分流 mfg1 / school1 / cloud / heartbeat 等 |

---

## 二、环境对照表

| 维度 | prod（生产，待建） | staging（预发） |
|------|-------------------|-----------------|
| 域名 | `mfg.ziwi.cn`（待建） | `mfg1.ziwi.cn` |
| 对外协议/端口 | HTTPS 443 | HTTPS 443 |
| 后端监听端口 | 待定 | `:8092` |
| 数据库端口 | 待定 | `:5434` |
| 数据库名 | 待定 | `mfg` |
| compose 文件 | `deploy/docker-compose.yml` | 同（通过 project name 区分） |
| env 文件 | `.env.prod` | `backend/.env`（含 CLOUD_JWKS_URL） |
| compose 项目名 | `mfg` | `mfg1` |
| 统一登录 | cloud.ziwi.cn RS256 JWKS 验签 | ✅ 已配置 |

---

## 三、技术栈与架构

- **后端**：Python 3.13 + FastAPI + SQLAlchemy (async) + PostgreSQL
- **前端**：Vue 3 + TypeScript + Vite（见 `frontend/`）
- **容器化**：Docker Compose（Python 镜像构建 + PostgreSQL 16 镜像）
- **数据库**：PostgreSQL 16（`mfg1-db` 容器，端口 5434）
- **统一认证**：信任 cloud.ziwi.cn 签发的 RS256 JWT，通过 JWKS 拉取公钥验签
- **代码仓库**：`sipon-wu/ziwi_mfg`（GitHub 私仓，deploy key 已配置）

---

## 四、环境依赖

| 类别 | 工具 / 版本 | 用途 |
|------|------------|------|
| 本地 | Python 3.13 | 后端开发与测试 |
| 本地 | Node.js + Vite | 前端构建 |
| 服务器 | Docker + Docker Compose | 容器编排 |
| 服务器 | nginx（host） + acme.sh | 域名分流 + 证书 |
| 服务器 | PostgreSQL 16 镜像 | 数据存储 |
| 网络 | SSH 到 `193.112.163.147` | 登录与部署 |
| 凭据 | `/opt/mfg1/backend/.env` | **勿硬编码到任何文档** |

---

## 五、标准工作操作流程（SOP）

### 前置
1. SSH 登录服务器：`ssh root@193.112.163.147`（凭据另附，勿入仓）。
2. 确保 GitHub deploy key 可用（`~/.ssh/ziwi_mfg_deploy_key`）。

### A. 仅后端改动
1. 本地修改代码 → 推送到 `ziwi_mfg` 仓库。
2. SSH 到服务器，`cd /opt/mfg1`。
3. `git pull` 拉取最新代码到 `/opt/mfg1/backend/`。
4. **重建容器**：`cd /opt/mfg1 && docker compose build --pull --no-cache && docker compose up -d`。
   - ⚠️ `--pull` 确保基础镜像为最新（旧 `python:3.13-slim` 的 pip 不认 cp313 wheel 会导致 cryptography/asyncpg 安装失败）。
   - ⚠️ `--no-cache` 确保依赖层重新安装（避免漏掉 `requirements.txt` 新增的包如 `openpyxl`）。
5. 验证 `curl -s http://127.0.0.1:8092/health` 返回 `{"status":"ok"}`。

### B. 含前端改动
6. 本地 `cd frontend && npm run build`。
7. `rsync -avz dist/ root@193.112.163.147:/var/www/mfg1.ziwi.cn/`。
8. 浏览器验证 `https://mfg1.ziwi.cn`。

### C. 数据库迁移
9. 如有 SQL 迁移文件（`backend/migrations/`），在容器启动后执行：
   ```bash
   docker exec mfg1-db psql -U mfg -d mfg -f /path/to/migration.sql
   ```

### D. 发布生产（待建）
10. staging 全量验证通过后，用相同 compose + 不同 `.env.prod` + 不同 project name 部署 prod。

### E. 回滚
11. `cd /opt/mfg1 && git checkout <previous-commit> && docker compose build --no-cache && docker compose up -d`。

---

## 六、必踩坑清单（mfg 已验证）

| # | 坑 | 正确做法 |
|---|----|---------|
| 1 | compose 构建路径 `build: ../backend` 在 `/opt/mfg1/` 下解析成不存在的 `/opt/backend` | 使用 `build: ./backend`（相对 compose 文件所在目录） |
| 2 | nginx `http2 on;` 在 nginx 1.18.0 报语法错误 | 改 `listen 443 ssl http2;` 单行写法 |
| 3 | CVM 缓存的旧基础镜像 pip 太旧，不认 cp313 wheel → `from versions: none` | `docker compose build --pull` 强制拉最新基础镜像 |
| 4 | `asyncpg==0.29.0` 在 Python 3.13 仅有 sdist 需 gcc 编译 | 改用 `asyncpg>=0.29.0`（共机 cloud-idp 验证） |
| 5 | `openpyxl` 未入 `requirements.txt` → `ModuleNotFoundError` | 所有新增依赖必须显式声明并本地验证 |
| 6 | nginx mfg1 block 未 reload → 外部 200 实为 school1 假响应 | 每次变更后 `nginx -t && systemctl reload nginx`，用 `curl -H 'Host: mfg1.ziwi.cn'` 区分验证 |
| 7 | `nohup` 构建进程随 SSH 断开被杀 | 用 `setsid bash -c '...' < /dev/null > /tmp/build.log 2>&1 &` |
| 8 | school1 默认 443 block 会接走未匹配域名 → 假成功 | mfg1 block 必须精确 `server_name mfg1.ziwi.cn;` |

---

## 七、交付检查清单（GATE）

- [ ] 后端改动已推送 GitHub 并在服务器 `git pull`
- [ ] `docker compose build --pull --no-cache`
- [ ] `docker ps` 确认 `mfg1-backend` 状态为 `Up`
- [ ] `curl http://127.0.0.1:8092/health` 返回 `{"status":"ok"}`
- [ ] `curl -H 'Host: mfg1.ziwi.cn' http://127.0.0.1/health` 与直接 8092 一致（nginx 分流正确）
- [ ] `curl https://mfg1.ziwi.cn/api/v1/auth/me` 返回 401（无 token）
- [ ] nginx/SSL/域名改动已登记 `deploy-log/` 等受影响方确认
- [ ] cloud JWKS 公钥可正常拉取（`docker logs mfg1-backend \| grep -i jwks` 无错误）

---

## 八、关联文档

- **部署变更生命周期**：`流程闭环/状态迁移与移交规范_V1.md` 第三章
- **cloud 接入契约**：`contracts/mfg接入cloud接口契约.md`
- **school 部署参照**：`runbooks/school双环境部署工作流.md`（同机 Go 栈参照）
