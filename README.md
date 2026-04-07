# OpenClaw 备份仓库

此仓库用于同步 OpenClaw 配置、记忆和智慧到 GitHub。

## 备份内容
- `openclaw.json` — 主配置
- `credentials/` — API 凭证
- `agents/` — Agent 配置
- `workspace/` — 工作区（记忆、SOUL、用户配置等）
- `cron/` — 定时任务

## 恢复方法
```bash
tar -xzf openclaw-backup_YYYY-MM-DD.tar.gz -C ~
```
