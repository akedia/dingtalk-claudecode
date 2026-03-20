# DingTalk（钉钉）Channel for Claude Code

通过 MCP 服务器将钉钉机器人连接到 Claude Code。

服务器使用 Stream 模式（WebSocket）连接钉钉，当用户给机器人发消息时，服务器将消息转发给当前 Claude Code 会话，并提供工具让 Claude 回复消息和发送文件。

## 前提条件

- [Bun](https://bun.sh) — MCP 服务器运行在 Bun 上。安装：`curl -fsSL https://bun.sh/install | bash`

## 快速开始

> 默认配对流程，适用于单用户私聊场景。群聊和多用户请参考下方说明。

**1. 创建钉钉应用和机器人**

前往[钉钉开放平台](https://open-dev.dingtalk.com/)，创建一个**企业内部应用**。

进入**机器人**页面，启用机器人能力。

在**机器人配置**中，将消息接收模式设置为 **Stream 模式**。无需公网域名。

**2. 获取凭证**

进入**凭证与基础信息**页面，找到：
- **AppKey**（Client ID）
- **AppSecret**（Client Secret）

**3. 安装插件**

以下是 Claude Code 命令 — 先运行 `claude` 启动会话。

```sh
claude plugin marketplace add https://github.com/akedia/dingtalk-claudecode
claude plugin install dingtalk@dingtalk-claudecode
```

确认 `/dingtalk:configure` 能 tab 补全。如果不能，重启会话。

**4. 配置凭证**

```
/dingtalk:configure <clientId> <clientSecret>
```

会将 `DINGTALK_CLIENT_ID=...` 和 `DINGTALK_CLIENT_SECRET=...` 写入 `~/.claude/channels/dingtalk/.env`。你也可以手动编辑该文件，或设置环境变量（环境变量优先级更高）。

**5. 用 channel 模式重启**

不加 `--channels` 服务器不会连接钉钉 — 退出当前会话，重新启动：

```sh
claude --channels plugin:dingtalk@dingtalk-claudecode
```

**6. 配对**

在钉钉中私聊你的机器人 — 它会回复一个 6 位配对码。在 Claude Code 会话中运行：

```
/dingtalk:access pair <配对码>
```

之后你的私聊消息就会到达 Claude。

**7. 锁定访问**

配对只是为了获取用户 ID。完成后切换到 `allowlist` 模式，防止陌生人触发配对码：

```
/dingtalk:access policy allowlist
```

## 访问控制

### 私聊策略

| 策略 | 说明 |
| --- | --- |
| `pairing`（默认） | 未知用户收到配对码，用 `/dingtalk:access pair <code>` 审批 |
| `allowlist` | 仅允许列表中的 staffId 私聊 |
| `disabled` | 不接受任何私聊 |

### 群聊支持

群聊按 conversationId 逐个启用：

```
/dingtalk:access group add <conversationId>
```

默认要求 @机器人 才响应。使用 `--no-mention` 可响应所有消息。

### 获取 ID

- **Staff ID**：配对时自动获取
- **Conversation ID**：将机器人加入群聊，@它，查看日志中的 conversationId

## 工具

| 工具 | 用途 |
| --- | --- |
| `reply` | 向聊天发送消息。参数：`chat_id` + `text`，可选 `files`（绝对路径数组）附加文件。自动分块长文本。优先使用 sessionWebhook 快速回复，超时后降级到 REST API。 |
| `send_file` | 向聊天发送文件。通过钉钉媒体 API 上传并发送文件消息。最大 20MB。 |

## 支持的消息类型

| 类型 | 处理方式 |
| --- | --- |
| 文本 | 直接转发 |
| 图片 | 下载到 `~/.claude/channels/dingtalk/inbox/`，路径包含在通知中 |
| 富文本 | 提取文本和图片 |
| 语音 | 包含语音识别文本（如果钉钉提供） |
| 视频 | 标记为 `[视频]` |
| 文件 | 显示文件名，下载到 inbox |
| 链接 | 提取标题、描述和 URL |

## 已知限制

### 同时只能有一个会话连接

钉钉 Stream 连接绑定到单个 Claude Code 会话。只有使用 `--channels` 启动的会话才能接收消息。这与官方的 Discord 和 Telegram channel 设计一致。

- **切换会话**：先关闭当前会话（`/exit` 或 Ctrl+C），再用 `--channels` 启动新会话
- **普通会话**：不加 `--channels` 的 `claude` 正常使用 — skill 命令（`/dingtalk:configure`、`/dingtalk:access`）仍然可用，但不会收到钉钉消息
- **多会话冲突**：如果同时有多个会话使用 `--channels`，钉钉会在连接间负载均衡分发消息，导致消息随机到达不同会话，应避免这种情况

### 工作目录固定

工作目录在会话启动时确定，**无法通过钉钉消息远程切换**。如果你不在电脑旁，无法切换到其他项目目录。

**变通方案**：在主目录启动会话（`cd ~ && claude --channels ...`），然后在钉钉消息中使用绝对路径（如"帮我看看 E:\project-a\src\main.ts"）。Claude 可以通过绝对路径访问任何位置的文件。

### 无消息历史和搜索

钉钉 Stream API 不提供消息历史和搜索功能。机器人只能看到实时到达的消息。如果 Claude 需要之前的上下文，会要求你粘贴或总结。

### 不支持消息编辑和表情回应

与 Discord 和 Telegram 不同，钉钉机器人 API 不支持编辑已发送的消息或添加表情回应。`reply` 工具只能发送新消息。

### 钉钉特有限制

- **文本长度**：sessionWebhook 单条消息约 2000 字符，超长回复自动拆分为多条消息
- **sessionWebhook 过期**：钉钉的 sessionWebhook（用于快速回复）约 35 分钟后过期，之后服务器降级到需要 access token 的 REST API
- **文件大小**：通过钉钉媒体 API 上传文件最大 20MB
- **Markdown 支持**：钉钉的 Markdown 渲染能力有限（不支持表格，格式化有限），服务器默认发送纯文本

## 架构

```
钉钉 App
    │
    │ Stream 模式（WebSocket）
    ▼
┌──────────────┐
│  server.ts   │  ← MCP 服务器（Bun）
│  dingtalk-   │
│  stream SDK  │
└──────┬───────┘
       │ stdio（MCP 协议）
       ▼
┌──────────────┐
│  Claude Code │
└──────────────┘
```

服务器使用：
- `dingtalk-stream` SDK 通过 WebSocket 接收消息
- 钉钉 REST API 发送主动消息
- `sessionWebhook` 快速内联回复（可用时优先使用）
- `@modelcontextprotocol/sdk` 与 Claude Code 进行 MCP 通信
