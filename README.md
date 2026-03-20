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
claude plugin marketplace add https://github.com/akedia/dingtalk-claudecode
claude plugin install dingtalk@dingtalk-claudecode
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
claude --channels plugin:dingtalk@dingtalk-claudecode
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

## Limitations

### One session at a time

The DingTalk Stream connection is bound to a single Claude Code session. Only the session started with `--channels` receives messages. This is the same design as the official Discord and Telegram channels.

- **Starting a new channel session**: Close the current session first (`/exit` or Ctrl+C), then start a new one with `--channels`.
- **Non-channel sessions**: Running `claude` without `--channels` works normally — skills (`/dingtalk:configure`, `/dingtalk:access`) are still available, but the bot won't receive messages.
- **Multiple sessions**: If multiple sessions connect with `--channels` simultaneously, DingTalk load-balances messages across connections unpredictably. Avoid this.

### Workspace is fixed per session

The working directory is set when the session starts and **cannot be changed remotely** via DingTalk messages. If you need to work on a different project while away from your PC, you must be at the terminal to restart the session in a new directory.

**Workaround**: Start the session in your home directory (`cd ~ && claude --channels ...`), then use absolute paths in DingTalk messages (e.g. "read E:\project-a\src\main.ts"). Claude can access any file via absolute paths regardless of the working directory.

### No message history or search

DingTalk's Stream API provides no message history or search capability. The bot only sees messages as they arrive in real time. If Claude needs earlier context, it will ask you to paste or summarize.

### No message editing or reactions

Unlike Discord and Telegram, DingTalk's bot API does not support editing sent messages or adding emoji reactions. The `reply` tool sends new messages only.

### DingTalk-specific constraints

- **Text chunk limit**: ~2000 characters per message via sessionWebhook. Longer replies are automatically split into multiple messages.
- **sessionWebhook expiry**: DingTalk's sessionWebhook (used for fast replies) expires after ~35 minutes. After expiry, the server falls back to the REST API which requires an access token.
- **File size limit**: 20MB per file upload via the DingTalk media API.
- **Markdown support**: DingTalk's markdown rendering is limited — no tables, limited formatting. The server sends plain text by default.

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
