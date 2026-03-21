---
name: claude-telegram-hooks
description: Set up automatic Telegram forwarding for Claude Code sessions. Forwards the user's terminal messages to Telegram with a custom prefix, and forwards Claude's responses automatically via Stop hook. Use this skill when the user wants to monitor or receive Claude Code conversations in Telegram, set up Telegram integration, or bridge their terminal session to a Telegram chat.
---

# Telegram Hooks Setup

This skill sets up two-way Telegram forwarding for Claude Code:
- **User messages** (UserPromptSubmit hook) → forwarded to Telegram with a configurable prefix
- **Claude responses** (Stop hook) → forwarded to Telegram automatically

## Step 1: Gather configuration

Ask the user for:
1. **Telegram Bot Token** — from [@BotFather](https://t.me/BotFather)
2. **Chat ID** — the numeric ID of the Telegram chat/user to forward to (tip: send a message to the bot and check `https://api.telegram.org/bot<TOKEN>/getUpdates`)
3. **User display name** — shown as prefix on user messages, e.g. `Oliver` → displays as `((Oliver))`

If the user doesn't have a bot token yet, guide them:
1. Open Telegram and message @BotFather
2. Send `/newbot` and follow the prompts
3. Copy the token BotFather gives you

## Step 2: Create the config file

Write `~/.claude/telegram-hooks.json` with the user's credentials:

```json
{
  "bot_token": "REPLACE_TOKEN",
  "chat_id": "REPLACE_CHAT_ID",
  "user_name": "REPLACE_NAME"
}
```

This file stays on the user's machine and is never shared.

## Step 3: Create the scripts

Create `~/.claude/scripts/` if it doesn't exist, then write both scripts.

### `~/.claude/scripts/telegram-forward.py` (Stop hook — Claude's responses)

```python
#!/usr/bin/env python3
"""Stop hook: forwards last assistant text response to Telegram."""
import sys, json, os, urllib.request

config_path = os.path.expanduser("~/.claude/telegram-hooks.json")
with open(config_path) as f:
    config = json.load(f)

BOT_TOKEN = config["bot_token"]
CHAT_ID = config["chat_id"]

try:
    data = json.load(sys.stdin)
    msg = data.get("last_assistant_message", "").strip()
    if not msg:
        sys.exit(0)

    payload = json.dumps({"chat_id": CHAT_ID, "text": msg[:4096]}).encode()
    req = urllib.request.Request(
        f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
        data=payload,
        headers={"Content-Type": "application/json"},
    )
    urllib.request.urlopen(req, timeout=10)

except Exception:
    pass  # Never block Claude on hook failure
```

### `~/.claude/scripts/telegram-user-forward.py` (UserPromptSubmit hook — user's messages)

```python
#!/usr/bin/env python3
"""UserPromptSubmit hook: forwards user's terminal message to Telegram with prefix."""
import sys, json, os, urllib.request

config_path = os.path.expanduser("~/.claude/telegram-hooks.json")
with open(config_path) as f:
    config = json.load(f)

BOT_TOKEN = config["bot_token"]
CHAT_ID = config["chat_id"]
PREFIX = f"(({config['user_name']}))"

try:
    data = json.load(sys.stdin)
    # UserPromptSubmit hook sends the message under the "prompt" key
    message = (data.get("prompt") or data.get("message") or "").strip()
    if not message:
        sys.exit(0)

    # Skip messages that originated from Telegram — already visible there
    if '<channel source="plugin:telegram:telegram"' in message:
        sys.exit(0)

    text = f"{PREFIX} {message}"
    payload = json.dumps({"chat_id": CHAT_ID, "text": text[:4096]}).encode()
    req = urllib.request.Request(
        f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
        data=payload,
        headers={"Content-Type": "application/json"},
    )
    urllib.request.urlopen(req, timeout=10)

except Exception:
    pass
```

## Step 4: Update settings.json

Read `~/.claude/settings.json` first, then merge in the following hooks (preserve existing settings):

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/scripts/telegram-user-forward.py",
            "timeout": 10,
            "async": true
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/scripts/telegram-forward.py",
            "timeout": 10,
            "async": true
          }
        ]
      }
    ]
  }
}
```

If hooks already exist in the file, merge carefully — don't overwrite existing hooks on other events.

## Step 5: Verify

Test the connection by running:

```bash
python3 -c "
import json, urllib.request
TOKEN = 'REPLACE_TOKEN'
CHAT_ID = 'REPLACE_CHAT_ID'
payload = json.dumps({'chat_id': CHAT_ID, 'text': 'Telegram hooks setup complete!'}).encode()
req = urllib.request.Request(f'https://api.telegram.org/bot{TOKEN}/sendMessage', data=payload, headers={'Content-Type': 'application/json'})
print(urllib.request.urlopen(req, timeout=10).read().decode())
"
```

If `"ok":true` appears in the output, the connection is working. Tell the user to check their Telegram for the test message.

## Key technical notes

- **UserPromptSubmit hook** sends data with a `prompt` key (not `message`) — the script handles both for compatibility
- **Telegram plugin compatibility**: messages from Telegram arrive wrapped in `<channel source="plugin:telegram:telegram">` tags — these are already visible in Telegram, so they are automatically skipped to avoid duplicate forwarding
- **Stop hook** receives `last_assistant_message` directly in stdin data — no need to read JSONL files
- Both hooks use `async: true` so they never block Claude's response
- Messages are truncated to 4096 characters (Telegram's limit)
