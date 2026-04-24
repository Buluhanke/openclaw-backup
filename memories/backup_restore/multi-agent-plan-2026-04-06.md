# 多机器人独立运行方案

> 整理时间：2026-04-06
> 目标：多个独立机器人，各有独立技能、独立模型、完全隔离、无上下文污染

---

## 📊 现状分析

### 你当前的配置
- **单Gateway进程** + **单Agent (main)**
- 模型：Ollama 本地模型（qwen2.5:14b、deepseek-r1:8b、llama3.2:latest）+ Minimax 云端
- 通道：微信、企业微信、QQ、Telegram、飞书
- 所有对话共享同一个 context window

### 核心问题
1. **上下文污染**：所有对话历史混在一起，容易产生"串台"
2. **技能冲突**：一个技能可能被多个场景复用，导致行为不一致
3. **模型统一**：所有任务都用同一个模型，资源分配不合理
4. **无隔离**：一个对话的 compaction summary 会影响其他对话

---

## 🅰️ 方案一：单网关多Agent模式（官方推荐）

### 架构
```
┌─────────────────────────────────────────────┐
│         OpenClaw Gateway (单进程)            │
├─────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Agent:   │  │ Agent:   │  │ Agent:   │  │
│  │ 小助     │  │ 1688助手 │  │ 社交Bot  │  │
│  │          │  │          │  │          │  │
│  │ 独立WS   │  │ 独立WS   │  │ 独立WS   │  │
│  │ 独立Session │  独立Session │  独立Session │
│  │ qwen2.5 │  │ deepseek │  │ llama3.2 │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

### 配置文件示例
```json
{
  "agents": {
    "defaults": {
      "model": "ollama/qwen2.5:14b",
      "workspace": "~/.openclaw/workspace-main"
    },
    "list": [
      {
        "id": "main",
        "name": "小助",
        "workspace": "~/.openclaw/workspace-main",
        "model": "ollama/qwen2.5:14b",
        "skills": ["wecom-*", "1688-product-search", "memory-*"]
      },
      {
        "id": "trading",
        "name": "1688交易助手",
        "workspace": "~/.openclaw/workspace-trading",
        "model": "ollama/deepseek-r1:8b",
        "skills": ["1688-*", "query-1688-product-detail"]
      },
      {
        "id": "social",
        "name": "社交管家",
        "workspace": "~/.openclaw/workspace-social",
        "model": "ollama/llama3.2:latest",
        "skills": ["qqbot-*", "discord"]
      }
    ]
  },
  "bindings": [
    { "agentId": "main", "match": { "channel": "openclaw-weixin" } },
    { "agentId": "trading", "match": { "channel": "wecom" } },
    { "agentId": "social", "match": { "channel": "qqbot" } }
  ]
}
```

### 优点
- ✅ 官方推荐，配置简单
- ✅ 共享 Gateway 资源，内存占用低
- ✅ 每个 agent 独立 workspace + session store
- ✅ 支持 per-agent 技能过滤

### 缺点
- ⚠️ **已知 Bug**：Issue #31110 报告了 session contamination 问题（已修复，但需确认版本）
- ⚠️ 一个 Gateway 挂全部挂
- ⚠️ 上下文窗口仍然共享 Gateway 进程内存

### 隔离保障
- 独立 `~/.openclaw/agents/<agentId>/sessions/`
- 独立 workspace 目录
- 独立 compaction history
- Per-agent 技能白名单

---

## 🅱️ 方案二：多实例完全隔离（最安全）

### 架构
```bash
# 进程1: 小助 (微信)
OPENCLAW_HOME=~/.openclaw-main \
OPENCLAW_PORT=18789 \
openclaw gateway

# 进程2: 1688助手 (企业微信)
OPENCLAW_HOME=~/.openclaw-trading \
OPENCLAW_PORT=18790 \
openclaw gateway

# 进程3: 社交管家 (QQ)
OPENCLAW_HOME=~/.openclaw-social \
OPENCLAW_PORT=18791 \
openclaw gateway
```

### 完整隔离
- 每个实例有独立：
  - `~/.openclaw-<name>/agents/`
  - `~/.openclaw-<name>/workspace/`
  - `~/.openclaw-<name>/sessions/`
  - `~/.openclaw-<name>/credentials/`
- 完全进程隔离，一个崩了不影响其他
- 独立端口，独立 model context window

### 优点
- ✅ **完全隔离**，无任何交叉污染可能
- ✅ 独立 Gateway 进程，互不干扰
- ✅ 可按需分配不同端口
- ✅ 适合稳定生产环境

### 缺点
- ⚠️ 资源占用更高（每个实例都跑一个 Gateway）
- ⚠️ 配置更复杂
- ⚠️ 需要进程管理（systemd/supervisor）
- ⚠️ 无法跨实例共享认证（如同一个企业微信 Bot）

### 适用场景
- 对隔离性要求极高
- 需要不同模型版本共存
- 需要独立扩缩容

---

## 🅲️ 方案三：混合架构（推荐生产使用）

### 设计原则
```
┌─────────────────────────────────────────────────────┐
│           Nginx / Traefik 反向代理                  │
│              (统一入口，按路径分发)                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│  /main/*    →  Gateway Main (小助)                   │
│              →  模型: qwen2.5:14b                   │
│              →  技能: 全能                          │
│                                                      │
│  /trading/* →  Gateway Trading (1688助手)           │
│              →  模型: deepseek-r1:8b                │
│              →  技能: 1688货源                      │
│                                                      │
│  /social/*  →  Gateway Social (社交管家)            │
│              →  模型: llama3.2:latest               │
│              →  技能: QQ/Discord                    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### 文件布局
```
~/.openclaw-main/          # 小助实例
~/.openclaw-trading/       # 1688助手实例
~/.openclaw-social/        # 社交管家实例
```

### Nginx 配置示例
```nginx
upstream main {
    server 127.0.0.1:18789;
}
upstream trading {
    server 127.0.0.1:18790;
}
upstream social {
    server 127.0.0.1:18791;
}

server {
    listen 80;
    
    location /main/ {
        proxy_pass http://main/;
        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    location /trading/ {
        proxy_pass http://trading/;
    }
    
    location /social/ {
        proxy_pass http://social/;
    }
}
```

### 优点
- ✅ 完全隔离 + 统一入口
- ✅ 可按业务类型独立扩展
- ✅ 配置灵活
- ✅ 适合微服务架构

### 缺点
- ⚠️ 架构复杂度较高
- ⚠️ 需要额外的进程管理
- ⚠️ 端口和路由配置较多

---

## 📋 实施路线图

### Phase 1：创建独立 Agent（单网关模式）
```bash
# 创建新的 agent workspace
openclaw agents add trading --workspace ~/.openclaw/workspace-trading
openclaw agents add social --workspace ~/.openclaw/workspace-social

# 配置 skills
# 在各自 workspace 的 AGENTS.md 中配置技能范围
```

### Phase 2：配置模型绑定
```json
{
  "agents": {
    "list": [
      { "id": "main", "model": "ollama/qwen2.5:14b" },
      { "id": "trading", "model": "ollama/deepseek-r1:8b" },
      { "id": "social", "model": "ollama/llama3.2:latest" }
    ]
  }
}
```

### Phase 3：配置 Channel Bindings
```json
{
  "bindings": [
    { "agentId": "main", "match": { "channel": "openclaw-weixin" } },
    { "agentId": "trading", "match": { "channel": "wecom" } },
    { "agentId": "social", "match": { "channel": "qqbot" } }
  ]
}
```

### Phase 4：验证隔离性
```bash
# 检查 session 目录是否分离
ls -la ~/.openclaw/agents/*/sessions/

# 验证 binding 配置
openclaw agents list --bindings

# 测试上下文污染
# 对 Agent A 说一个秘密，对 Agent B 说同样的秘密
# 确认它们不会互相知道
```

---

## 🔧 关键配置项

### Per-Agent 技能白名单
```json
{
  "agents": {
    "list": [
      {
        "id": "trading",
        "skills": {
          "allow": ["1688-*", "query-*"],
          "deny": ["wecom-*", "qqbot-*"]
        }
      }
    ]
  }
}
```

### Per-Agent Session 配置
```json
{
  "agents": {
    "list": [
      {
        "id": "trading",
        "session": {
          "reset": {
            "idleMinutes": 60
          }
        }
      }
    ]
  }
}
```

### Compaction 隔离
```json
{
  "agents": {
    "list": [
      {
        "id": "trading",
        "compaction": {
          "mode": "safeguard",
          "reserveTokens": 20000
        }
      }
    ]
  }
}
```

---

## ⚠️ 已知问题与修复

### Issue #31110: Session Contamination
- **状态**：已修复（2026-03-02 合并 PR #31188）
- **修复版本**：建议升级到最新稳定版
- **临时方案**：使用多实例隔离（方案二/三）

### 版本检查
```bash
openclaw --version
# 确保 >= 2026.3.x
```

---

## 📊 推荐方案选择

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 个人使用，简单需求 | 方案一 | 配置简单，官方支持 |
| 生产环境，高隔离要求 | 方案二 | 完全隔离，最安全 |
| 微服务架构，多团队 | 方案三 | 隔离 + 统一入口 |

### 你的情况推荐
考虑到你的需求（1688货源监控 + 社交管理），推荐：

**方案一（短期）** → **方案二/三（长期）**

先用单网关多 agent 模式快速上手，后续根据负载和稳定性需求升级到多实例隔离。

---

## 🚀 下一步行动

1. **确认版本**：`openclaw --version` 确保已修复 session contamination bug
2. **创建新 Agent**：
   ```bash
   openclaw agents add trading --workspace ~/.openclaw/workspace-trading
   ```
3. **配置 Binding**：将企业微信绑定到 trading agent
4. **测试隔离**：验证两个 agent 的 session 完全分离

---

*方案整理：小助 ✨ | 2026-04-06*
