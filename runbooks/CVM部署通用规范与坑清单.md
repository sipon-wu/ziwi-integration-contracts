# CVM 服务器部署通用规范与坑清单（跨项目）

> 适用：所有部署在 CVM `193.112.163.147`（腾讯轻量云）的项目 —— school / mfg / ecms / cloud / heartbeat。
> 目的：把「谁有权部署、部署副本怎么对齐、踩过哪些坑、完成门槛是什么」统一沉淀，**避免每个项目从零踩**。
> 配套：`school双环境部署工作流.md`（school 专属双环境）、`../流程闭环/状态迁移与移交规范_V1.md`、`../测试标准/`。

---

## 一、权限与密钥模型（最重要，2026-07-12 厘清）

- **CVM root SSH**（`/root/.ssh/authorized_keys`）现持有两把 key：
  - `sipon@192.168.0.103` —— 本机 Mac，长期持有，原唯一 key
  - `workbuddy` —— 2026-07-12 新增（ed25519，无密码短语），**授予 workbuddy CVM root SSH，使其能自行部署 mfg**
- **结论：CVM 部署权限长期只在 Mac 侧。**
  - cloud / heartbeat 之所以稳，**不是因为不需要 key**，而是它们本就由 Mac 侧部署/维护（systemd + alembic 自动迁移，或 `/opt/heartbeat` 自带 docker-compose），部署权与工具链都在 key 持有方手里。
  - mfg 之乱，根因是**开发在 workbuddy（离 Mac）**，而部署这步（`git pull` → `rsync` → `docker rebuild`）没人 owning、无脚本，掉进 handoff 缝隙，paper-green 一路藏到 N5 才炸。
  - 所以「workbuddy 做不了我做的对齐」本质是**部署权限/归属问题**，非能力问题。
- **github 远端 key 是只读 deploy key**：在 CVM 上只能 `git pull`，`git push` 会报 `The key you are authenticating with has been marked as read only`。任何 push 必须从这台 Mac（持有写 key）执行。
- **纪律**：新增 CVM 部署权限（加 key）属授权操作，须用户明确指令；workbuddy key 即由此授权加入。

---

## 二、各项目部署事实速查

| 项目 | 路径 | 运行方式 | 部署注意 |
|------|------|---------|---------|
| school | `/opt/zhiwei/code` | docker compose（`zhiwei` / `zhiwei-staging`） | 后端本地编译；后端改动须 rsync 到 `/opt/zhiwei/code/backend` 再 `up --build`；staging 必须 `--env-file .env.staging` |
| mfg | `/opt/ziwi/mfg` | docker compose（`deploy/docker-compose.yml`，service `mfg-backend`，`container_name: mfg1-backend`） | 见第三节「部署副本分离」+ `deploy.sh` |
| ecms | `/var/www/ecms` + 后端 systemd | 前端 dist + FastAPI systemd（崩溃自拉） | 前端 `base` 根路径；后端 systemd 托管 |
| cloud | `/opt/cloud-idp` | FastAPI + systemd + **alembic 自动迁移** | 短链路、自动迁移，最稳 |
| heartbeat | `/opt/heartbeat` | 复用 cloud 同后端 + 独立 docker-compose / nginx 反代 | 无独立代码，独立域名反代 |

---

## 三、mfg 部署副本分离模式（核心坑，2026-07-12）

- git 跟踪 `backend/`；但运行容器 `mfg1-backend` 的 build context 是 `deploy/backend/`（整个 **untracked** 的手动 rsync 副本）。
- **`git pull` 只更新 `backend/`，不碰 `deploy/backend/`** → 若只 pull 不 rsync，容器跑的还是旧副本，「对齐了个寂寞」。
- 固化脚本 **`deploy.sh`**（已推 github main `c21bc65`）已解决，步骤：
  1. `rm -f backend/migrations/add_work_order_status_logs_tenant_id.sql`（游离迁移曾挡 pull）
  2. `git checkout -- . && git pull --ff-only`
  3. `rsync -a --exclude=Dockerfile --exclude=.git backend/ deploy/backend/`（保留 deploy 优化 Dockerfile）
  4. `cd deploy && docker rm -f mfg1-backend && docker compose up -d --no-deps --build mfg-backend`
  5. `sleep 12` + health 验证（端口 `127.0.0.1:8092/health`）
- **用法（workbuddy 在 CVM 上）**：`cd /opt/ziwi/mfg && git pull origin main && ./deploy.sh`
- **绝不可动 `mfg1-db`**（数据卷 `mfg_pgdata`）；`--no-deps` 保证不重建 db。

---

## 四、通用坑清单（跨项目，避免重复踩）

| # | 坑 | 正确做法 |
|---|----|---------|
| G1 | `git pull` 只更新 git 跟踪目录，容器跑的是另一份部署副本（如 mfg 的 `deploy/backend/`）→ 对齐无效 | 部署副本必须 rsync 同步；固化成 `deploy.sh`，**禁止只 pull** |
| G2 | `docker compose up` 默认按 `depends_on` 重建依赖服务（如 db），触发容器名冲突、搞挂 staging | 重建单服务用 `docker compose up -d --no-deps <service>`；`container_name` 固定时先 `docker rm -f <container_name>` |
| G3 | CVM 上 github key 只读，误在 CVM `git push` 报 `read only` | push 只能从 Mac（写 key）执行；CVM 仅 pull |
| G4 | 验证探针路径错：打 `/wms/locations` 实则 `/api/v1/wms/locations`，误报 404「端点缺失」 | 探针必须按真实路由前缀打；任何「404 缺失」先核对源码路由前缀再下结论 |
| G5 | rsync 排除规则误伤源码：`--exclude='server'` 排除所有含 `server` 路径分量的目录（含 `cmd/server/`），新端点 404 | exclude 锚定传输根：`--exclude='/server'`；改完 `cmd/server/` 后确认排除不漏 |
| G6 | 手搓 `docker compose` 漏 `--env-file`/`--build`/`--no-deps` → 连错库、没重编、网络没挂 | 部署只走固化脚本（`deploy.sh` / `deploy.sh staging`），禁手搓 |
| G7 | 验证「接口 200 即绿」放行，隐性闭环（对账/过账/流水）漏验 | 把「业务对账探针」升为交付闸门；涉及「账」的模块没跑过收/发/出对账不算 done |
| G8 | 跨环境责任断裂：workbuddy 开发 paper-green 即报 done，问题溢出到 CVM 才救火 | 共享仓 SOP 为共同契约；进入 `Ready-for-QA` 前强制读部署坑清单 + GATE；状态进 `STATUS.md` |

> school 专属坑（env-file、nginx `/ai/` 抢匹配、`listen_addresses`、Docker Hub 被墙、证书等）见 `school双环境部署工作流.md` 第六节。

---

## 五、部署完成门槛（Definition of Done，任何项目强制）

1. 本地 build 通过
2. **真 token 实测该端点**（非只 curl 200；前端须真浏览器 E2E，渲染级 smoke 不能替交互验证）
3. 全回归（API 套件 + Playwright 全链路）PASS
4. **部署对齐脚本跑通**（`deploy.sh` / `deploy.sh staging` 成功，容器用新镜像）
5. **业务对账探针通过**（涉及账/库存/流水者必跑）
6. 若有 nginx / SSL / 域名改动 → 登记 `deploy-log/` 且受影响方确认（流程闭环规范第三章）

缺任一项不算 done。

---

## 六、与其他文档关系

- school 专属双环境：`school双环境部署工作流.md`
- 流程闭环 / 移交：`../流程闭环/状态迁移与移交规范_V1.md`
- 测试标准：`../测试标准/AI测试Agent约束指南.md` + `QA_从严回测用例集_V1.md`
- mfg 移交与待办：`../mfg_移交物_预发布对齐与待办_20260712.md`
