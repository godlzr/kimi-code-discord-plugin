# Discord bridge for Kimi Code CLI

Connect a Discord bot to Kimi Code CLI with an MCP server. Ported from the
[Claude Code Discord plugin](https://github.com/anthropics/claude-code) —
same access-control model (pairing, allowlists, guild-channel opt-in).

Paired DMs are delivered via **push**: an inbound message wakes a running
session immediately, through Kimi Code CLI's generic
`notifications/kimi/channel` MCP notification. This only reaches the agent
when the CLI's `mcp-channel` experimental flag is enabled (see
[Enabling push](#enabling-push) — it's off by default). Without it, or for
group channels (which never push), or when the session wasn't running when
the message arrived, `check_new_messages` is the reliable fallback — the
agent (or you) asks what came in since the last check. Everything else —
pairing, allowlists, mention gating, reply/react/edit/attachments — is
generic Discord-bot logic and works the same way regardless of push.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with
  `curl -fsSL https://bun.sh/install | bash`.
- Kimi Code CLI installed (`kimi`).

## Quick Setup

**1. Create a Discord application and bot.**

Go to the [Discord Developer Portal](https://discord.com/developers/applications)
and click **New Application**. Give it a name.

Navigate to **Bot** in the sidebar. Give your bot a username.

Scroll down to **Privileged Gateway Intents** and enable **Message Content
Intent** — without this the bot receives messages with empty content.

**2. Generate a bot token.**

Still on the **Bot** page, scroll up to **Token** and press **Reset
Token**. Copy the token — it's only shown once. Hold onto it for step 5.

**3. Invite the bot to a server.**

Discord won't let you DM a bot unless you share a server with it.

Navigate to **OAuth2** → **URL Generator**. Select the `bot` scope. Under
**Bot Permissions**, enable:

- View Channels
- Send Messages
- Send Messages in Threads
- Read Message History
- Attach Files
- Add Reactions

Integration type: **Guild Install**. Copy the **Generated URL**, open it,
and add the bot to any server you're in.

> For DM-only use you technically need zero permissions — but enabling
> them now saves a trip back when you want guild channels later.

**4. Install the plugin.**

Start a session: `kimi`. Then, inside the session:

```
/plugins install https://github.com/godlzr/kimi-code-discord-plugin.git
```

(or a local path — `/plugins install /path/to/kimi-code-discord-plugin` — if
you cloned it yourself; a `.zip` URL also works.)

The plugin's MCP server (`discord`) is enabled by default — check with
`/plugins` if you want to confirm. Run `/reload` or `/new` afterward to
apply.

**5. Give the server the token.**

In your Kimi Code session:

```
/skill:discord-configure MTIz...
```

Writes `DISCORD_BOT_TOKEN=...` to `~/.kimi-code/channels/discord/.env`. You can
also write that file by hand, or set the variable in your shell
environment — shell takes precedence.

> To run multiple bots on one machine (different tokens, separate
> allowlists), point `DISCORD_STATE_DIR` at a different directory per
> instance.

**6. Restart the session.**

The MCP server reads the token once at boot — exit and start a new `kimi`
session so it picks it up.

**7. Pair.**

DM your bot on Discord — it replies with a pairing code. In your Kimi Code
session:

```
/skill:discord-access pair <code>
```

**8. Check it.**

```
check discord
```

Kimi should call `list_chats` / `check_new_messages` and show you the DM.
Ask it to reply and confirm the message shows up on Discord.

**9. Lock it down.**

Pairing is for capturing IDs. Once you're in, switch to `allowlist` so
strangers don't get pairing-code replies:

```
/skill:discord-access policy allowlist
```

## Enabling push

By default Kimi Code CLI only picks up new Discord DMs when you (or the
agent) call `check_new_messages`. To have a paired DM wake a running
session immediately, start `kimi` with the `mcp-channel` experimental flag
on:

```sh
KIMI_CODE_EXPERIMENTAL_MCP_CHANNEL=1 kimi
```

(or set it in your shell profile / the `[experimental]` section of Kimi
Code's config, or use the `KIMI_CODE_EXPERIMENTAL_FLAG=1` master switch).
This is a CLI-side flag, not a plugin setting — the plugin always *sends*
the push notification when a paired DM arrives; without the flag, Kimi
Code CLI silently ignores it and you're back to poll-only. Group channels
never push, flag or not — `check_new_messages` is the only way to see
those.

> **Rendering the pushed message still needs one more thing.** The engine
> wakes on push already, but showing the pushed text as its own message
> (instead of only through the model's own reply) requires
> [MoonshotAI/kimi-code#2432](https://github.com/MoonshotAI/kimi-code/pull/2432),
> which isn't merged yet. Until it lands, either wait for the PR, or build
> Kimi Code CLI from
> [godlzr/kimi-code@feat/mcp-channel-tui-render](https://github.com/godlzr/kimi-code/tree/feat/mcp-channel-tui-render)
> to get it now.

## Access control

Same shape as the Claude Code plugin's `access.json`:

```json
{
  "dmPolicy": "pairing",
  "allowFrom": ["<senderId>", ...],
  "groups": { "<channelId>": { "requireMention": true, "allowFrom": [] } },
  "pending": { "<code>": { "senderId": "...", "createdAt": 0, "expiresAt": 0 } },
  "mentionPatterns": ["@mybot"]
}
```

Managed entirely through the `discord-access` skill — see
`skills/discord-access/SKILL.md` for the full command list (`pair`, `deny`,
`allow`, `remove`, `policy`, `group add/rm`, `set`).

IDs are Discord **snowflakes** (numeric — enable Developer Mode, right-click
→ Copy ID). Guild channels are opt-in per channel ID, not per server.

## Tools exposed to the agent

| Tool | Purpose |
| --- | --- |
| `list_chats` | List reachable chats (paired DM users + opted-in group channels) with their `chat_id`. |
| `check_new_messages` | Poll for messages received since the last check, across all reachable chats. The reliable fallback when push isn't enabled/supported, and the only way to see group channels. First sighting of a chat only seeds the read position. |
| `fetch_messages` | Recent history from one chat regardless of read position (oldest-first, capped at 100). |
| `reply` | Send to a chat. `chat_id` + `text`, optionally `reply_to` (message ID) for native threading and `files` (absolute paths) for attachments — max 10 files, 25MB each. Auto-chunks at 2000 chars. |
| `react` | Add an emoji reaction to any message by ID. |
| `edit_message` | Edit a message the bot previously sent. |
| `download_attachment` | Download attachments from a specific message to `~/.kimi-code/channels/discord/inbox/`. |

Paired DMs push automatically when the flag is on (see
[Enabling push](#enabling-push)). For group channels, or whenever push
isn't in play, "relaying Discord messages during a session" means the
agent calls `check_new_messages` at natural checkpoints — see
`skills/discord-messaging/SKILL.md`.

## Attachments

Not auto-downloaded. `check_new_messages`/`fetch_messages` list each
attachment's name, type, and size; call `download_attachment(chat_id,
message_id)` when you actually want the file.

## Push vs. Claude Code's plugin

Claude Code's discord plugin declares an experimental MCP capability
(`claude/channel`) that its host treats specially: an inbound Discord
message becomes a notification Claude Code injects into the active
conversation as if the user had just spoken, plus a matching
`claude/channel/permission` capability that relays tool-permission prompts
back out to Discord. Kimi Code CLI has its own, unrelated equivalent:
`notifications/kimi/channel`, a generic MCP notification any server can
send — this plugin uses it for paired DMs, tagged with a distinct
`mcp_channel` origin (not treated as if the user typed it) rather than a
permission-relay capability. It's gated behind the CLI's `mcp-channel`
experimental flag (see [Enabling push](#enabling-push)), so until that
flag graduates to on-by-default, `check_new_messages` remains the
dependable path — and the only one for group channels, which push never
covers.
