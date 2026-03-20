# DingTalk (钉钉)

Connect a DingTalk bot to your Claude Code with an MCP server.

The MCP server connects to DingTalk via Stream Mode (WebSocket) and provides tools to Claude to reply and send files. When you message the bot, the server forwards the message to your Claude Code session.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.

## Quick Setup
> Default pairing flow for a single-user DM bot. See below for groups and multi-user setups.

**1. Create a DingTalk application and bot.**

Go to the [DingTalk Developer Platform](https://open-dev.dingtalk.com/) and create an Enterprise Internal Application (企业内部应用).

Navigate to **Robot** (机器人) and enable the bot capability.

Under **Bot Configuration** (机器人配置), set the message receiving mode to **Stream Mode** (Stream 模式). No public domain required.

**2. Get your credentials.**

Navigate to **Credentials** (凭证与基础信息) and find:
- **AppKey** (Client ID)
- **AppSecret** (Client Secret)

**3. Install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

```
/plugin install dingtalk@claude-plugins-official
/reload-plugins
```

Check that `/dingtalk:configure` tab-completes. If not, restart your session.

**4. Give the server the credentials.**

```
/dingtalk:configure <clientId> <clientSecret>
```

Writes `DINGTALK_CLIENT_ID=...` and `DINGTALK_CLIENT_SECRET=...` to `~/.claude/channels/dingtalk/.env`. You can also write that file by hand, or set the variables in your shell environment — shell takes precedence.

**5. Relaunch with the channel flag.**

The server won't connect without this — exit your session and start a new one:

```sh
claude --channels plugin:dingtalk@claude-plugins-official
```

**6. Pair.**

DM your bot on DingTalk — it replies with a 6-character pairing code. In your assistant session:

```
/dingtalk:access pair <code>
```

Your next DM reaches the assistant.

**7. Lock it down.**

Pairing is for capturing IDs. Once you're in, switch to `allowlist` so strangers don't get pairing-code replies. Ask Claude to do it, or `/dingtalk:access policy allowlist` directly.

## Access control

### DM Policies

- **pairing** (default): Unknown users get a pairing code. Approve with `/dingtalk:access pair <code>`.
- **allowlist**: Only staff IDs in the allowlist can DM the bot.
- **disabled**: No DMs accepted.

### Group Support

Groups are opt-in by conversation ID:

```
/dingtalk:access group add <conversationId>
```

By default, the bot only responds when @mentioned in groups. Use `--no-mention` to respond to all messages.

### Getting IDs

- **Staff ID**: Automatically captured during pairing
- **Conversation ID**: Add bot to group, @mention it, check logs

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`, optionally `files` (absolute paths) for attachments. Auto-chunks long text. Uses sessionWebhook when available for fast delivery, falls back to REST API. |
| `send_file` | Send a file to a chat. Uploads via DingTalk media API and sends as a file message. Max 20MB. |

Inbound messages are forwarded immediately — DingTalk Stream Mode receives them via WebSocket.

## Message Types

The bot handles these inbound message types:
- **Text**: Plain text messages
- **Picture**: Downloaded to `~/.claude/channels/dingtalk/inbox/` and path included in notification
- **Rich Text**: Text + embedded images
- **Audio**: Speech recognition text included when available
- **Video**: Marked as `[视频]`
- **File**: File name shown, downloaded to inbox
- **Link**: Title, text, and URL extracted

## No history or search

DingTalk's Stream API exposes **neither** message history nor search. The bot only sees messages as they arrive. If the assistant needs earlier context, it will ask you to paste or summarize.

## Architecture

```
DingTalk (钉钉)
    │
    │ Stream Mode (WebSocket)
    ▼
┌──────────────┐
│  server.ts   │  ← MCP server (Bun)
│  dingtalk-   │
│  stream SDK  │
└──────┬───────┘
       │ stdio (MCP protocol)
       ▼
┌──────────────┐
│  Claude Code │
└──────────────┘
```

The server uses:
- `dingtalk-stream` SDK for receiving messages via WebSocket
- DingTalk REST API for sending proactive messages
- `sessionWebhook` for fast inline replies (when available)
- `@modelcontextprotocol/sdk` for MCP communication with Claude Code
