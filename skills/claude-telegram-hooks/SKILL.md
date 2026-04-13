---
name: claude-telegram-hooks
description: Set up automatic Telegram forwarding for Claude Code sessions. Forwards the user's terminal messages to Telegram with a custom prefix, forwards Claude's responses automatically via Stop hook, sends attention alerts via Notification hook, and sends context compaction alerts via PreCompact hook. Use this skill when the user wants to monitor or receive Claude Code conversations in Telegram, set up Telegram integration, or bridge their terminal session to a Telegram chat.
---

# Telegram Hooks Setup

This skill sets up two-way Telegram forwarding for Claude Code:
- **User messages** (UserPromptSubmit hook) → forwarded to Telegram with a configurable prefix
- **Claude responses** (Stop hook) → forwarded to Telegram automatically
- **Attention alerts** (Notification hook) → sent when Claude needs user input or permission
- **Context compaction alerts** (PreCompact hook) → sent when token limit is reached

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

Create `~/.claude/scripts/` if it doesn't exist, then write all three scripts.

### `~/.claude/scripts/telegram-forward.py` (Stop hook — Claude's responses)

```python
#!/usr/bin/env python3
"""Stop hook: forwards last assistant text response to Telegram."""
import sys, json, os, re, urllib.request

config_path = os.path.expanduser("~/.claude/telegram-hooks.json")
with open(config_path) as f:
    config = json.load(f)

BOT_TOKEN = config["bot_token"]
CHAT_ID = config["chat_id"]

# System tags that should not be forwarded to Telegram
SYSTEM_TAG_PATTERNS = [
    r"<task-notification>.*?</task-notification>",
    r"<system-reminder>.*?</system-reminder>",
    r"\(\(Oliver\)\)\s*<[^>]+>.*",
]

try:
    data = json.load(sys.stdin)
    msg = data.get("last_assistant_message", "").strip()
    if not msg:
        sys.exit(0)

    # Strip system tags
    for pattern in SYSTEM_TAG_PATTERNS:
        msg = re.sub(pattern, "", msg, flags=re.DOTALL)
    msg = msg.strip()

    if not msg:
        sys.exit(0)

    payload = json.dumps({"chat_id": CHAT_ID, "text": msg[:4096]}).encode()
    req = urllib.request.Request(
        f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
        data=payload,
        headers={"Content-Type": "application/json"},
    )
    urllib.request.urlopen(req, timeout=10)

    # Mark pending Telegram reply as answered
    pending_path = os.path.expanduser("~/.claude/.telegram-pending-reply")
    if os.path.exists(pending_path):
        os.remove(pending_path)

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

    # If message came from Telegram, save message_id as pending reply state
    if '<channel source="plugin:telegram:telegram"' in message:
        import re
        m = re.search(r'message_id="(\d+)"', message)
        if m:
            pending_path = os.path.expanduser("~/.claude/.telegram-pending-reply")
            with open(pending_path, "w") as pf:
                pf.write(m.group(1))
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

### `~/.claude/scripts/precompact-telegram-check.py` (PreCompact hook — unanswered message guard)

If a Telegram message came in but Claude's session compresses before sending a reply, this script fires first and notifies the user so the message isn't silently lost.

```python
#!/usr/bin/env python3
"""PreCompact hook: warn via Telegram if there's an unanswered Telegram message."""
import os, json, urllib.request

config_path = os.path.expanduser("~/.claude/telegram-hooks.json")
pending_path = os.path.expanduser("~/.claude/.telegram-pending-reply")

if not os.path.exists(pending_path):
    exit(0)

try:
    with open(pending_path) as f:
        message_id = f.read().strip()

    with open(config_path) as f:
        config = json.load(f)

    BOT_TOKEN = config["bot_token"]
    CHAT_ID = config["chat_id"]

    text = f"⚠️ 세션 압축 직전 — 텔레그램 메시지 #{message_id}에 아직 답변이 전송되지 않았어요. 다음 세션에서 이어서 답변할게요."
    payload = json.dumps({"chat_id": CHAT_ID, "text": text}).encode()
    req = urllib.request.Request(
        f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
        data=payload,
        headers={"Content-Type": "application/json"},
    )
    urllib.request.urlopen(req, timeout=10)

except Exception:
    pass
```

### `~/.claude/scripts/telegram-notify.py` (Notification hook — attention alerts)

Only fires when actual user decision is required (e.g. permission prompts). Generic "waiting for input" messages after task completion are filtered out to avoid noise.

```python
#!/usr/bin/env python3
"""Notification hook: Claude가 실제 사용자 결정이 필요할 때만 텔레그램으로 알림."""
import sys, json, os, urllib.request

config_path = os.path.expanduser("~/.claude/telegram-hooks.json")
with open(config_path) as f:
    config = json.load(f)

BOT_TOKEN = config["bot_token"]
CHAT_ID = config["chat_id"]

# 일반 대기 메시지 (단순 작업 완료 후 입력 대기) — 알림 불필요
GENERIC_IDLE_PATTERNS = [
    "claude is waiting for your input",
    "waiting for your input",
    "waiting for input",
]

try:
    data = json.load(sys.stdin)
    msg = data.get("message", "").strip()

    # 메시지가 없거나 일반 대기 메시지면 전송하지 않음
    if not msg:
        sys.exit(0)
    if any(pattern in msg.lower() for pattern in GENERIC_IDLE_PATTERNS):
        sys.exit(0)

    payload = json.dumps({
        "chat_id": CHAT_ID,
        "text": f"🔔 {msg[:4000]}"
    }).encode()
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
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/scripts/precompact-telegram-check.py; echo '{\"message\": \"⚡ 토큰 한도 도달 - 컨텍스트 압축 중\"}' | python3 ~/.claude/scripts/telegram-notify.py",
            "timeout": 10,
            "async": true
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/scripts/telegram-notify.py",
            "timeout": 10,
            "async": true
          }
        ]
      }
    ],
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
- **`claude -p` subprocess trap**: when code calls `claude -p "..."` as a subprocess (e.g. to invoke an LLM for analysis), that spawns a new Claude Code session with its own hooks. The `UserPromptSubmit` hook fires on that session and forwards the prompt text to Telegram — including any long automated prompts like portfolio analysis queries. **Fix**: always pass `--bare` flag: `claude --bare -p "..."`. The `--bare` flag skips all hooks (and LSP, plugins, etc.), so the subprocess runs silently without triggering `UserPromptSubmit`.
- **Stop hook** receives `last_assistant_message` directly in stdin data — no need to read JSONL files
- **Notification hook** receives a `message` key in stdin — falls back to "⏳ Claude가 입력 대기 중입니다." if empty
- **PreCompact hook** first runs `precompact-telegram-check.py` to warn if there's an unanswered Telegram message, then uses `echo` to inject a fixed JSON compaction alert
- **Pending reply tracking**: when a Telegram message arrives, `UserPromptSubmit` saves the message_id to `~/.claude/.telegram-pending-reply`. The `Stop` hook deletes this file after forwarding a response. If `PreCompact` fires while the file still exists, the user gets a warning that the message was not answered before compaction.
- All hooks use `async: true` so they never block Claude's response
- Messages are truncated to 4096 characters (Telegram's limit)

## Hook event coverage

| Event | Hook | When it fires |
|-------|------|---------------|
| UserPromptSubmit | telegram-user-forward.py | User sends a message in terminal; saves message_id if from Telegram |
| Stop | telegram-forward.py | Claude finishes a response; clears pending reply state |
| Notification | telegram-notify.py | Claude needs user attention (permission prompt, input wait) |
| PreCompact | precompact-telegram-check.py + telegram-notify.py (inline) | Warns if unanswered Telegram message exists, then sends compaction alert |
