# OpenClaw IT Team 部署指南

本文档记录了将 openclaw-it-team 项目部署到 OpenClaw 的完整过程，包括配置、问题排查和解决方案。

## 目录

1. [概述](#概述)
2. [准备工作](#准备工作)
3. [Telegram Bot 创建](#telegram-bot-创建)
4. [配置文件](#配置文件)
5. [配对与授权](#配对与授权)
6. [群组配置](#群组配置)
7. [问题排查](#问题排查)
8. [最终配置](#最终配置)

---

## 概述

openclaw-it-team 是一个多 Agent 协作项目，包含 4 个角色：

| Agent | 角色 | 职责 |
|-------|------|------|
| PM | 项目经理 | 需求澄清、任务派发、进度跟进 |
| RD | 工程师 | 开发、自测、提测 |
| QA | 测试工程师 | 功能测试、Bug 报告 |
| CE | 鼓励师 | 情绪协同、氛围调节 |

---

## 准备工作

### 1. 安装 OpenClaw

```bash
npm install -g openclaw
```

### 2. 初始化配置目录

```bash
mkdir -p ~/.openclaw
```

### 3. 创建 Agent 工作目录

需要为每个 Agent 创建独立的工作目录，包含 8 个标准文件：

```
workspace-pm/
├── AGENTS.md
├── BOOTSTRAP.md
├── HEARTBEAT.md
├── IDENTITY.md
├── MEMORY.md
├── SOUL.md
├── TOOLS.md
└── USER.md
```

---

## Telegram Bot 创建

### 1. 创建 Bot

在 Telegram 中与 @BotFather 对话：

1. 发送 `/newbot`
2. 输入 Bot 名称（如 DiyuhuoPMBot）
3. 输入用户名（必须以 bot 结尾，如 DiyuhuoPMBot）
4. 复制 token

### 2. 创建多个 Bot

本项目需要 5 个 Bot：

| Bot 名称 | 用途 |
|----------|------|
| DiyuhuoBot | 原有 Bot，用于 main/stock-insights |
| DiyuhuoPMBot | PM 专用 |
| DiyuhuoRDBot | RD 专用 |
| DiyuhuoQABot | QA 专用 |
| DiyuhuoCEBot | CE 专用 |

### 3. 关闭隐私模式（重要！）

**必须操作**：在 @BotFather 中关闭每个 Bot 的 Group Privacy：

1. `/mybots`
2. 选择 Bot
3. Bot Settings → Group Privacy
4. **关闭** Enable group privacy

> ⚠️ 关闭隐私后，必须将 Bot 从群组中移除并重新添加，更改才能生效。

### 4. 添加 Bot 到群组

1. 创建 Telegram 群组（如 DiyuhuoITTeam）
2. 将 4 个 Bot 添加到群里
3. 设置为管理员（可选，但推荐）

---

## 配置文件

### openclaw.json 结构

```json
{
  "models": {
    "providers": {
      "minimax-cn": {
        "apiKey": "your-api-key",
        "models": [...]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "minimax-cn/MiniMax-M2.5" },
      "workspace": "/Users/dong/.openclaw/workspace"
    },
    "list": [
      { "id": "pm", "workspace": "...", "agentDir": "...", ... },
      { "id": "rd", ... },
      { "id": "qa", ... },
      { "id": "ce", ... }
    ]
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "defaultAccount": "DiyuhuoBot",
      "dmPolicy": "pairing",
      "groups": {
        "*": { "requireMention": false }
      },
      "groupPolicy": "open",
      "accounts": {
        "DiyuhuoBot": { "botToken": "...", ... },
        "DiyuhuoPMBot": { "botToken": "...", ... },
        "DiyuhuoRDBot": { "botToken": "...", ... },
        "DiyuhuoQABot": { "botToken": "...", ... },
        "DiyuhuoCEBot": { "botToken": "...", ... }
      }
    }
  },
  "bindings": [
    { "agentId": "pm", "match": { "channel": "telegram", "accountId": "DiyuhuoPMBot" } },
    { "agentId": "rd", "match": { "channel": "telegram", "accountId": "DiyuhuoRDBot" } },
    { "agentId": "qa", "match": { "channel": "telegram", "accountId": "DiyuhuoQABot" } },
    { "agentId": "ce", "match": { "channel": "telegram", "accountId": "DiyuhuoCEBot" } }
  ]
}
```

### 关键配置说明

| 配置项 | 说明 |
|--------|------|
| `dmPolicy: "pairing"` | 私信需要配对授权 |
| `groupPolicy: "open"` | 群组消息不限制发送者 |
| `groups.*.requireMention: false` | 不需要 @ 即可响应（CE 推荐）|
| `defaultAccount` | 默认 Bot 账号 |

---

## 配对与授权

### 1. 私信配对

首次使用需要配对：

1. 在 Bot 中发送消息
2. 获取配对码
3. 批准配对

```bash
# 查看配对请求
openclaw pairing list telegram

# 批准配对
openclaw pairing approve telegram <CODE>
```

### 2. 群组权限

确保用户 ID 在白名单中，或者使用 `groupPolicy: "open"`。

---

## 群组配置

### 工作原理

当 Bot 关闭隐私模式后，所有群消息都会被 Bot 接收。OpenClaw 根据 `bindings` 配置决定哪个 Agent 响应。

### 绑定方式

有两种方式：

#### 方式1：accountId 绑定（推荐）

```json
{
  "agentId": "pm",
  "match": {
    "channel": "telegram",
    "accountId": "DiyuhuoPMBot"
  }
}
```

- `@DiyuhuoPMBot` → 触发 PM
- `@DiyuhuoRDBot` → 触发 RD
- 依此类推

#### 方式2：peer group 绑定

```json
{
  "agentId": "pm",
  "match": {
    "channel": "telegram",
    "peer": {
      "kind": "group",
      "id": "-5123821087"
    }
  }
}
```

- 群内所有消息都触发指定 Agent
- 需要配合 `mentionPatterns` 或 requireMention 使用

---

## 问题排查

### 问题1：Bot 收不到群消息

**原因**：Telegram 隐私模式默认开启

**解决**：
1. 在 @BotFather 中关闭 Group Privacy
2. 将 Bot 从群移除并重新添加

### 问题2：@Bot 没有反应

**可能原因**：

1. **Bot 不在群里** - 确认 Bot 已被添加到群
2. **隐私模式未关闭** - 参见问题1
3. **配对未完成** - 私信 Bot 完成配对
4. **bindings 配置错误** - 检查 bindings 配置

### 问题3：消息被路由到错误的 Agent

**原因**：群组 bindings 配置了 `peer.group`，导致所有消息都路由到同一个 Agent

**解决**：移除 peer group 绑定，只使用 accountId 绑定

### 问题4：gateway connect failed

**原因**：Gateway 进程停止

**解决**：
```bash
openclaw gateway restart
```

### 问题5：accounts.default is missing

**原因**：缺少默认账号配置

**解决**：
```json
"defaultAccount": "DiyuhuoBot"
```

### 问题6：群消息被静默丢弃

**原因**：`groupPolicy: "allowlist"` 但 allowFrom 为空

**解决**：
```json
"groupPolicy": "open"
```

---

## 最终配置

### 目录结构

```
~/.openclaw/
├── openclaw.json           # 主配置
├── agents/
│   ├── main/
│   ├── pm/
│   ├── rd/
│   ├── qa/
│   ├── ce/
│   └── stock-insights/
├── workspace-pm/
├── workspace-rd/
├── workspace-qa/
├── workspace-ce/
└── credentials/
    ├── telegram-*.json   # 配对凭证
    └── ...
```

### 常用命令

```bash
# 启动 Gateway
openclaw gateway start

# 重启 Gateway
openclaw gateway restart

# 查看状态
openclaw status

# 查看通道列表
openclaw channels list

# 查看配对列表
openclaw pairing list telegram

# 批准配对
openclaw pairing approve telegram <CODE>

# 查看日志
openclaw logs --follow
```

### 群组使用

在 DiyuhuoITTeam 群中：

| 命令 | 触发 Agent |
|------|-----------|
| `@DiyuhuoPMBot 你好` | PM |
| `@DiyuhuoRDBot 你好` | RD |
| `@DiyuhuoQABot 你好` | QA |
| `@DiyuhuoCEBot 你好` | CE |

---

## 注意事项

1. **隐私模式**：每次修改 Bot 的隐私设置后，必须将 Bot 移出群并重新添加
2. **Token 安全**：不要将 Bot Token 提交到代码仓库
3. **配置合并**：如果已有配置，部署新配置时注意合并而不是替换
4. **模型配置**：根据实际可用的模型修改 `agents.defaults.model.primary`

---

## 参考资料

- [OpenClaw Telegram 文档](https://docs.openclaw.ai/zh-CN/channels/telegram)
- [openclaw-it-team 项目](https://github.com/openclaw/skills/tree/main/skills/halfmoon82/coding-team-setup)

---

*本文档最后更新于 2026-03-15*
