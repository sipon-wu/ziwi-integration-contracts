# 知微 AI 教学助手 — school / school1 双环境部署工作流

> 目的：让任何接入团队（school / mfg / ecms 等）照此流程部署，**不重复踩已知坑**。
> 配套：流程闭环规范第三章（部署变更生命周期，nginx/SSL 改动必须登记 `deploy-log/` 并等受影响方确认）；
> QA_从严回测用例集（发布前 staging 全量回归）。

---

## 一、方案目的

| 目标 | 说明 |
|------|------|
| **安全发布** | 所有改动先在 `staging(school1)` 验证，通过后再发 `prod(school)`，避免未验证代码上生产 |
| **镜像隔离** | docker-compose 项目名 `zhiwei`（prod） vs `zhiwei-staging`（staging），防止镜像互相覆盖 |
| **数据隔离** | 库 `zhiwei`（prod） vs `zhiwei_staging`（staging），互不影响；prod 数据清洗需明确指令 |
| **证书策略** | prod `school.ziwi.cn` 走 Let's Encrypt；staging `school1.ziwi.cn` 复用通配符 `*.ziwi.cn_ecc` |
| **域名分流** | host nginx 按 `server_name` 分流多站点（school / school1 / www / cloud 等） |

---

## 二、环境对照表

| 维度 | prod（生产） | staging（预发） |
|------|--------------|-----------------|
| 域名 | `school.ziwi.cn` | `school1.ziwi.cn` |
| 对外协议/端口 | HTTPS 443 | HTTPS 443（原仅 80，2026-07-09 补 443） |
| 后端监听端口 | `:8080` | `:8081` |
| 数据库名 | `zhiwei` | `zhiwei_staging` |
| compose 文件 | `docker-compose.prod.yml` | `docker-compose.staging.yml` |
| env 文件 | `/opt/zhiwei/code/deploy/.env` | `.env.staging` |
| compose 项目名 | `zhiwei` | `zhiwei-staging` |
| 前端 DOCROOT | `/var/www/school.ziwi.cn` | `/var/www/school1.ziwi.cn` |
| 前端快照回滚 | `/var/www/.deploy_snapshots/prod/` | `/var/www/.deploy_snapshots/staging/` |

> ⚠️ **staging 不带 `--env-file .env.staging` 会连到 prod 的 `zhiwei` 库**（已踩坑）。任何 staging 操作必须带 env-file。

---

## 三、架构与组件

- **服务器**：`193.112.163.147`（腾讯轻量云），host nginx 做域名分流；Docker Compose 管理容器。
- **容器**（compose 管理）：`zhiwei-demo-api` / `ai` / `db` / `redis` / `nginx`(已停用) / `reset-cron`。
- **数据库**：PostgreSQL 16（替代 pgvector，`listen_addresses=*`）。
- **缓存**：Redis。
- **前端**：Vite 构建 → 产物推到服务器 `/var/www/<域名>`。
- **后端**：Go，在服务器本地 `/opt/zhiwei/code/backend` **现编译**（不是构建镜像时拉代码）。
- **部署脚本**：`code/deploy/deploy.sh <prod|staging>`（本地 vite build 推 dist → docker compose up）。
- **证书**：acme.sh 签发的 Let's Encrypt；`school1` 复用 `/root/.acme.sh/*.ziwi.cn_ecc/` 通配符。
- **统一登录**：school 侧已接 cloud（RS256 JWKS 验签 + HS256 会话），通过 `CloudJWKSURL` 配置；staging 已验证 P0/P1（commit `d3b4518`、`68e383f`）。

---

## 四、环境依赖 / 工具清单

| 类别 | 工具 / 版本 | 用途 |
|------|------------|------|
| 本地构建 | Node.js + Vite | 前端 `npm run build` |
| 后端改动的本地侧 | Go 工具链 + rsync | 后端改动须 rsync 源码到服务器再编译 |
| 服务器 | Docker + Docker Compose | 容器编排 |
| 服务器 | nginx（host） + acme.sh | 域名分流 + 证书 |
| 服务器 | PostgreSQL 16 镜像、Redis 镜像 | 数据 / 缓存 |
| 网络 | SSH 到 `193.112.163.147` | 登录与部署 |
| 数据库访问 | SSH 隧道（本地端口 → 容器 PG） | 只读查数据（见第七节） |
| 凭据 | 取自 `/opt/zhiwei/code/deploy/.env(.staging)` | **勿硬编码到任何文档/代码** |

---

## 五、标准工作操作流程（SOP）

### 前置
1. SSH 登录服务器：`ssh root@193.112.163.147`（凭据另附，勿入仓）。
2. 本地拉取最新代码。

### A. 仅前端改动
3. 本地 `npm run build`（前端 `base` 已为根路径；曾误配 `/demo/ecms/` 已修正）。
4. 执行 `./code/deploy/deploy.sh staging` → 推 dist 到 `/var/www/school1.ziwi.cn`，再 `docker compose up`。
5. 浏览器验证 `https://school1.ziwi.cn`。

### B. 含后端改动（关键！）
6. **后端改动必须先把源码 rsync 到服务器** `/opt/zhiwei/code/backend`，否则容器用的是旧源码编译。
7. `docker compose -f docker-compose.staging.yml --env-file .env.staging up --build`
   ⚠️ 漏 `--env-file` 会连错库。
8. 验证。

### C. 发布生产
9. staging 全量用例（QA_从严回测用例集）通过后，再 `deploy.sh prod` 或对应 `up --build`。
10. **prod 数据清洗 / 迁移需用户明确说"清"等指令，不得擅自操作。**

### D. 回滚
11. 前端快照回滚：`/var/www/.deploy_snapshots/staging/`（prod 同理 prod 目录）。
12. 后端：回退代码重新 `up --build`，或复用旧镜像。

---

## 六、必踩坑清单（避免重复踩）

> school 专属坑见下；**跨项目 CVM 通用坑**（部署副本分离、只读 github key、容器名冲突、探针路径纪律、完成门槛）见 `CVM部署通用规范与坑清单.md`。

| # | 坑 | 正确做法 |
|---|----|---------|
| 1 | 后端改了代码，staging 没生效 | 后端容器本地编译，**先 rsync 源码到服务器 `/opt/zhiwei/code/backend`** 再 `up --build` |
| 2 | staging 连到 prod 库 | `up` 必须带 `--env-file .env.staging` |
| 3 | Nginx `/ai/` 规则抢在 `/api/v1` 前匹配，API 失效 | 删掉 `/ai/` 规则，只用 Go 代理 |
| 4 | 系统 nginx 占 80 端口，容器起不来 | 停 system nginx，用 host nginx 分流；Docker nginx 改为辅助 |
| 5 | `listen_addresses` 默认 localhost，容器间连不上 DB | 启动加 `-c listen_addresses=*` |
| 6 | Docker Hub 被墙，镜像拉不下来 | 用腾讯云镜像源（仅 `library/*` 可用）；pgvector 替换为 `postgres:16` |
| 7 | `DATABASE_URL` env 缺失，RAG 模块连不上 DB | 补 `postgresql://` 连接串到 env |
| 8 | school1 原仅 80，Chrome 自动 HTTPS 升级被全机 443 接走显示知微云，真人进不去 | 已建 `school1.ziwi.cn.ssl`（listen 443，复用 `*.ziwi.cn_ecc`，反代 `:8081`） |
| 9 | 误清 prod 数据 | prod 数据清洗需明确指令；staging/prod 库名隔离 |
| 10 | cloud 对接 staging 验证不充分 | 已验证 P0/P1（commit `d3b4518`、`68e383f`）；接 cloud 用 `CloudJWKSURL` 配置 |
| 11 | rsync 排除规则误伤源码：`--exclude='server'` 排除所有含 `server` 路径分量的目录（含 `cmd/server/`），新端点 404 | exclude 锚定传输根 `--exclude='/server'`；改完 `cmd/server/` 后确认排除规则不漏（详见 `CVM部署通用规范与坑清单.md` G5） |

---

## 七、数据库访问（SSH 隧道，只读）

- 隧道：`本地 15432` → staging 容器 `172.19.0.3:5432`。
- 凭据取自 `/opt/zhiwei/code/deploy/.env.staging`（`DB_USER` / `DB_PASSWORD` / `DB_NAME`），**勿写入本文档或代码**。
- prod 同理，但需明确授权。

---

## 八、交付检查清单（GATE，发布前自审）

- [ ] staging 已部署并人工/用例验证通过
- [ ] 后端改动已 rsync 到服务器
- [ ] `up` 已带正确 `--env-file`
- [ ] 若有 nginx / SSL / 域名改动 → 已登记 `deploy-log/` 且受影响方确认（流程闭环规范第三章）
- [ ] prod 发布前 staging 全量用例（QA_从严回测用例集）PASS
- [ ] 未擅自清洗 prod 数据

---

## 九、与其他文档关系

- **部署变更生命周期**：`../流程闭环/状态迁移与移交规范_V1.md` 第三章 —— nginx/SSL 改动强制闭环
- **测试左移与用例**：`../测试标准/AI测试Agent约束指南.md` + `QA_从严回测用例集_V1.md`
- **接口契约（接入 cloud）**：`../contracts/` 或 `../测试标准/school接入cloud接口契约模板.md`
- **跨项目 CVM 通用坑与权限模型**：`CVM部署通用规范与坑清单.md`（部署副本分离、只读 github key、容器名冲突、探针路径纪律、完成门槛 DoD）
