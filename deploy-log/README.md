# deploy-log — 部署变更互盯登记（A 类）

> **用途**：任何团队改动 **nginx / SSL 证书 / 域名分流 / 反向代理** 等影响多方的部署，必须在此登记，等**受影响方确认**才闭环。
> 这是"谁动了部署、会不会坑到别人"的变更广播，全员平权共写（school / mfg / ecms 等）。
> 生命周期定义见：`../流程闭环/状态迁移与移交规范_V1.md` 第三章。

## 生命周期（闭环）

```
[拟部署 Draft] → [已部署 Deployed] → [受影响方确认 Affected-Confirmed] → [已闭环 Closed]
                     │                                              │
                     └── 异常/踩到他人 ──▶ [回滚 Rollback] → Draft
```

| 状态 | 含义 | 退出条件 |
|------|------|---------|
| Draft | 计划改动，登记 | 实际部署完成 |
| Deployed | 配置生效 + 自验通过 | 所有受影响方确认无踩踏 |
| Affected-Confirmed | 受影响团队签字确认 | 超时未异议 → 自动 Closed |
| Closed | 终态 | — |
| Rollback | 出现跨服务故障 | 回滚完成 → 重新 Draft |

## 强制规则（防冲突）

1. **只追加**：每次部署在 `变更记录.md` 表尾加一行，**不改历史行**。
2. **改前先 `git pull`**：拉最新再追加，减少合并冲突。
3. **状态必更新**：状态列必须随生命周期推进（Deployed / Affected-Confirmed / Closed / Rollback）。
4. **nginx/SSL 改动必须等受影响方在"确认方"列签字确认**，才置 `Closed`——这是互盯的强制闭环。
5. 涉及 cloud / heartbeat 等公共服务的改动，确认方须包含 mfg。

## 登记表

实际追加在 **`变更记录.md`**（不要改本文件）。
| 2026-07-10 | mfg | 新增 `mfg1.ziwi.cn` nginx 443 vhost，反代 `127.0.0.1:8092`，复用 `*.ziwi.cn_ecc` 通配符证书 | 共享 nginx `sites-enabled/`，DNS 新增 A 记录 | nginx + 域名 | Deployed → Affected-Confirmed | school / ecms |
