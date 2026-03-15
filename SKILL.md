---
name: openclaw-it-team-deploy
description: 为 openclaw-it-team 仓库生成和维护部署型工作流。用于把当前仓库的多 Agent 配置安装到 OpenClaw、本地化 `openclaw.json`、填充模型与渠道账号配置，并完成启动前验收。支持飞书、Telegram、Slack、Discord 等多种渠道。
---

# OpenClaw IT Team Deploy Skill

本 Skill 用于部署当前仓库这套多 Agent 配置。

适用场景：
- 用户要求"部署 / 安装 / 配置 / 跑起来"当前仓库
- 用户要求把仓库中的 `agents/*` 安装到本机 OpenClaw
- 用户要求生成或修改 `~/.openclaw/openclaw.json` 以接入这套 `PM -> RD -> QA -> CE` 团队
- 用户要求接入渠道 Bot（飞书/Telegram/Slack/Discord）、模型提供商、Agent 路由或工作目录

这不是传统 Web 项目部署。当前仓库的“部署”本质上是：
1. 把仓库内的 Agent Workspace 文件安装到 OpenClaw 目录
2. 在用户现有配置基础上合并新内容（参考 Telegram 模板）
3. 填入模型与渠道账号配置
4. 启动 OpenClaw 并验收 Agent 路由是否正常

## 先读哪些文件

默认只读这些：
- `README.md`：理解仓库目标与角色分工
- `docs/openclaw.telegram.json`：Telegram 部署模板（默认）
- `docs/openclaw.json`：飞书部署模板（用户指定飞书时使用）
- `agents/pm/*`、`agents/rd/*`、`agents/qa/*`、`agents/ce/*`：要安装的实际 Agent 文件

仅在需要解释配置字段时，再读：
- `docs/openclaw.config.docs.md`

## 部署原则

- 优先替用户直接完成落盘、替换、校验，不要只给口头步骤
- 不要假设用户机器是 `/root`；把模板中的 `/root/.openclaw/...` 改成用户真实 Home 目录
- 不要在没拿到真实凭据前伪造“部署成功”
- 如果 OpenClaw 启动命令在当前机器不可知，先探测本机可用命令；不要编造命令
- 若仓库内容与目标目录已存在，先比对再覆盖，避免无提示破坏用户现有配置

## 标准目标目录

默认建议部署到用户主目录下（若用户指定其他目录，以用户指定为准）：

```text
~/.openclaw/
├── openclaw.json
├── workspace/
├── workspace-pm/
├── workspace-rd/
├── workspace-qa/
└── workspace-ce/
```

注意：仓库里的源目录是 `agents/pm` 这种结构，部署目标应映射到角色工作目录（如 `~/.openclaw/workspace-pm`），不要再部署到 `~/.openclaw/agents/pm/agent` 这种旧路径。

## 标准工作流

### 1. 先确认部署前提

至少确认这些信息：
- OpenClaw 已安装，或者用户允许你检查本机 OpenClaw 命令
- 目标部署目录：默认 `~/.openclaw`（用户指定其他目录时按用户指定执行）
- 模型提供商与模型 ID
- 使用的渠道类型（**推荐 Telegram**，也支持飞书/Slack/Discord/其他）
- 若使用 Telegram，是否已提供四个 Bot 的 `botToken`（推荐）
- 若使用飞书，是否已提供四个 Bot 的 `appId` 和 `appSecret`
- 若使用其他渠道，是否已提供对应的凭据

如果用户没有明确指定渠道，**默认使用 Telegram**。

如果用户只是想先把配置落地、不立即联通渠道，可以先写模板，但必须明确哪些占位符仍未完成。

### 2. 安装 Agent 文件

把仓库内四个角色目录复制到目标目录：

```text
repo/agents/pm/* -> ~/.openclaw/workspace-pm/
repo/agents/rd/* -> ~/.openclaw/workspace-rd/
repo/agents/qa/* -> ~/.openclaw/workspace-qa/
repo/agents/ce/* -> ~/.openclaw/workspace-ce/
```

要求：
- 每个目标目录都应包含 8 个标准文件：`AGENTS.md`、`BOOTSTRAP.md`、`HEARTBEAT.md`、`IDENTITY.md`、`MEMORY.md`、`SOUL.md`、`TOOLS.md`、`USER.md`
- 缺文件时，不要继续宣称部署完成

### 3. 合并配置（重要！）

**必须保留用户的现有配置！** 不允许直接替换用户的 `openclaw.json`。

#### 3.1 读取现有配置

1. 读取用户现有的 `~/.openclaw/openclaw.json`
2. 备份现有配置（如 `openclaw.json.bak`）
3. 在现有配置基础上**合并**新内容

#### 3.2 合并规则

| 配置项 | 处理方式 |
|--------|---------|
| `models.providers` | **保留**，只补充不存在的 provider |
| `agents.defaults.model` | **保留**，不覆盖用户的主模型 |
| `agents.defaults.workspace` | **保留**，不覆盖 |
| `agents.list` | **追加**（在现有 agent 列表后添加 pm/rd/qa/ce）|
| `subagents.allowAgents` | **追加**（如 main 现有 allowAgents: [a,b]，变成 [a,b,pm,rd,qa,ce]）|
| `tools.agentToAgent` | **合并**，添加到 allow 列表 |
| `bindings` | **追加**（添加新的 pm/rd/qa/ce bindings）|
| `channels` | **保留**现有渠道配置，新增 telegram-bot-* 账号 |
| `gateway` | **保留**，不覆盖 |
| `session` | **保留**，不覆盖 |
| `messages` | **保留**，不覆盖 |
| `commands` | **保留**，不覆盖 |

#### 3.3 禁止操作

- ❌ 不要删除用户的现有 agents（如 main、stock-insights）
- ❌ 不要替换用户的模型配置
- ❌ 不要删除用户的现有渠道配置
- ❌ 不要替换 bindings

#### 3.4 Telegram Bot 账号合并

如果用户已有 Telegram 配置：
- 在现有 `channels.telegram` 下添加新账号
- 保持现有 botToken 不变
- 新账号填写用户提供的 token

```json
// 合并后示例
"channels": {
  "telegram": {
    "botToken": "用户现有...",
    "groups": { ... },
    "accounts": {
      "telegram-bot-pm": { "botToken": "用户新提供..." },
      "telegram-bot-rd": { "botToken": "用户新提供..." },
      ...
    }
  }
}
```

### 4. 模型配置规则

#### 4.1 自动检测（推荐）

优先探测用户本机已注册的模型：
1. 读取用户现有的 `openclaw.json` 中的 `models.providers`
2. 从已注册的模型中智能匹配：
   - **最强推理**：适合 Architect、DevOps、Security → 匹配 `/opus/i`、`/4.6/i`、`/o1/i`
   - **代码专长**：适合 Frontend、Backend、Code Artisan → 匹配 `/codex/i`、`/5-codex/i`
   - **均衡型**：适合 PM、QA、Tech Writer → 匹配 `/sonnet/i`、`/haiku/i`
3. 为不同 Agent 角色分配最适合的模型

#### 4.2 手动指定

若用户明确指定模型：
- 同步修改 `models.providers`
- 同步修改 `agents.defaults.model.primary`
- 同步修改 `agents.defaults.models` 中与主模型对应的键
- 确保 provider/id 的引用一致，不要只改一处

#### 4.3 默认模板

如果无法检测到用户模型，使用默认配置：
```json
"primary": "zai/glm-5"
```

模型选择建议：
- PM、QA、CE：均衡型模型
- RD（开发）：代码专长模型
- 全部使用同一模型也可工作

### 5. 渠道配置规则

当前仓库设计为四个 Agent 绑定四个渠道账号。**默认使用 Telegram**。

#### 5.1 Telegram 配置（推荐/默认）

若用户使用 Telegram：
- `telegram-bot-pm`、`telegram-bot-rd`、`telegram-bot-qa`、`telegram-bot-ce`
- 填写 `botToken`
- Telegram 不需要 `requireMention` 配置，每个 Bot 会接收所有发送到该 Bot 的消息

#### 5.2 飞书配置

若用户使用飞书：
- `feishu-bot-pm`、`feishu-bot-rd`、`feishu-bot-qa`、`feishu-bot-ce`
- 填写 `appId` 和 `appSecret`
- `ce` 默认 `requireMention: false`，其余角色默认 `requireMention: true`

#### 5.3 Slack 配置

若用户使用 Slack：
- `slack-bot-pm`、`slack-bot-rd`、`slack-bot-qa`、`slack-bot-ce`
- 填写 `botToken` 和 `appToken`

#### 5.4 Discord 配置

若用户使用 Discord：
- `discord-bot-pm`、`discord-bot-rd`、`discord-bot-qa`、`discord-bot-ce`
- 填写 `token`

#### 5.5 通用处理规则

- 有真实凭据时，替换对应占位值
- 没有真实凭据时，不要写假值；保留占位符并明确标记为未完成
- `ce`（鼓励师）若渠道支持，建议设置 `requireMention: false`（全局监听氛围）
- `bindings` 中 `agentId` 和 `accountId` 必须一一对应

### 6. 启动前检查

至少做这些检查：
- 目标 `openclaw.json` 是合法 JSON
- `agents.list[*].workspace` 指向的目录都存在
- `agents.list[*].agentDir` 与对应 `workspace-*` 目录一致
- 四个角色工作目录（`workspace-pm/rd/qa/ce`）都含 8 个标准文件
- 若填写了模型 provider，关键字段 `baseUrl`、`api`、`models` 结构完整
- 若填写了渠道账号（如飞书 appId/appSecret、Telegram botToken 等），凭据没有残留明显占位符

### 7. 启动与验收

优先顺序：
1. 先探测本机 OpenClaw 启动方式
2. 使用本机真实可用命令启动，不要编造
3. 启动后检查网关或日志，确认配置被正确加载

可接受的验收信号：
- OpenClaw 进程正常启动，无配置解析错误
- `openclaw.json` 被加载，无 `workspace/agentDir` 路径缺失报错
- 网关监听 `127.0.0.1:18789` 或用户指定端口
- 渠道（如飞书/Telegram/Slack/Discord）启用时，各账号绑定未报错
- **用户的原有 agent（如 main、stock-insights）仍然正常工作**

如果本机没有 OpenClaw 可执行命令，或用户未提供安装方式：
- 明确说明"配置已落地，但运行验证未完成"
- 不要把"文件已写入"表述成"服务已部署成功"

## 修改当前仓库时的输出要求

如果用户要求你"完善这个仓库，让后续部署更方便"，优先做这些事：
- 保持 `docs/openclaw.telegram.json` 作为默认模板来源，`docs/openclaw.json` 作为飞书备选
- 保持 `README.md` 面向人类阅读，`SKILL.md` 面向代理执行
- 在 `SKILL.md` 中只写执行流程、判断条件、路径映射、校验规则
- 不要把长篇配置字段解释重复写进 `SKILL.md`；需要时引用 `docs/openclaw.config.docs.md`

## 完成任务后的汇报格式

完成部署类任务后，汇报应至少包含：
- 部署到了哪个目录
- 修改了哪些关键文件
- 哪些配置已完成
- 哪些占位符仍待用户提供
- 是否已完成实际启动验证
- 若未验证，卡在哪里
- **用户原有配置是否已保留**（如 main agent、stock-insights、现有渠道等）
