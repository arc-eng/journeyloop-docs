---
title: Dotfile IPC Pattern
description: How sandboxed coaching agents communicate with the operator — file-based messaging via JSON request files.
---

# :material-file-code-outline: Dotfile IPC Pattern

**Since:** March 2026 · **Status:** Production

How sandboxed coaching agents request side-effects from the operator — without messaging tools, sockets, or shared state.

---

## Why It Exists

Coaching agents run in sandboxed Docker containers. They have no access to messaging tools, the OpenClaw CLI, or any external network service. They *can* write files to their workspace.

The dotfile IPC pattern exploits this: an agent declares what it wants by writing a JSON file; the operator monitor detects it, executes the request, and deletes the file.

```
Agent workspace                       Operator monitor
─────────────────                     ─────────────────────────────────────
.deliver-file.request.json   →→→→→    detect → deliver file → delete dotfile
.telegram-ui.request.json    →→→→→    detect → send poll/buttons → delete dotfile
```

Agents declare *intent*. The operator makes it happen. No agent needs to know its own Telegram ID or account name — the monitor resolves those from `.credentials.json`.

---

## Request File Format

All request files follow the naming convention `.<action>.request.json` and are written to the agent's workspace root.

Every request is a JSON object. Fields:

| Field | Required | Description |
|-------|----------|-------------|
| `type` | ✅ | Handler to invoke: `deliver-file` or `telegram-ui` |
| `account` | optional | OpenClaw account. Defaults to the value in `.credentials.json`. |
| `target` | optional | Destination (`telegram:<numeric_id>`). Defaults to `.credentials.json`. |
| `created_at` | optional | ISO timestamp for audit logging |

If `account` and `target` are omitted, the monitor reads them from the agent's `.credentials.json` — agents provisioned after March 2026 don't need to pass these at all.

The dotfile is **always deleted** after processing — success or failure.

---

## Supported Types

### `deliver-file`

Send a file to the coach on Telegram.

**Dotfile:** `.deliver-file.request.json`

```json
{
  "type": "deliver-file",
  "file": "progress-report.pdf",
  "caption": "Here's your monthly progress report! 📄",
  "created_at": "2026-03-13T06:00:00Z"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `file` | ✅ | Filename relative to the agent's workspace root |
| `caption` | optional | Message text sent with the file |

The file must exist in the workspace at the time the dotfile is written.

**Companion skill:** `file-delivery-proxy` — call `journeyloop companion deliver-file <filename>` in the sandbox. No `--to` flag needed.

---

### `telegram-ui`

Send a Telegram UI element: a poll or a message with inline buttons.

**Dotfile:** `.telegram-ui.request.json`

#### Poll

```json
{
  "type": "telegram-ui",
  "action": "poll",
  "question": "How are you feeling going into this week?",
  "options": ["Energized", "Steady", "Stretched thin"],
  "anonymous": false
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `action` | ✅ | `"poll"` |
| `question` | ✅ | Poll question text |
| `options` | ✅ | Array of option strings |
| `anonymous` | optional | Default: `false` |

#### Message with inline buttons

```json
{
  "type": "telegram-ui",
  "action": "message",
  "text": "Ready to review your goals?",
  "buttons": [
    { "text": "Yes, let's go", "callback_data": "goals_yes" },
    { "text": "Not right now", "callback_data": "goals_no" }
  ]
}
```

**Companion skill:** `telegram-ui` — call `journeyloop companion send-poll` or `journeyloop companion send-message` with `--button`.

---

## How the Monitor Works

`file-delivery-monitor.sh` runs on the GCP VM, watching all agent workspaces via inotify. On a file-create event matching `*.request.json`:

1. Reads the file contents
2. Routes to the appropriate handler (`deliver-file` or `telegram-ui`)
3. Resolves `account` + `target` from the request or from `.credentials.json`
4. Executes the action via the OpenClaw `message` tool
5. Deletes the request file

**Reliability design** (since #477):

- No `set -e` — a failed delivery doesn't kill the monitor
- Logs appended (`>>`) — a restart doesn't wipe history
- Crontab watchdog: checked every 5 minutes and at `@reboot`

---

## Delivery Target Resolution

Before #476, agents had to pass `--to <telegram_id>` explicitly. Now:

- The reconciler writes `telegram_account` and `telegram_chat_id` into `.credentials.json` at provision time
- The monitor reads these automatically — no `--to` needed in the sandbox
- Explicit `account`/`target` fields in the request JSON still work as an override (useful for manual operator sends)

---

## CLI Split: `journeyloop` vs `jl-operator`

!!! warning "Two CLIs, two environments"
    These CLIs are NOT interchangeable.

| CLI | Runs in | Purpose |
|-----|---------|---------|
| `journeyloop` | Companion sandbox (Docker) | Data access, companion IPC (`deliver-file`, `send-poll`, `send-message`) |
| `jl-operator` | GCP VM host | Reconcile, sync, deploy-skills, routines |

Companion IPC commands (`deliver-file`, `send-poll`, `send-message`) belong in the `journeyloop` CLI — they're sandbox-side. The `jl-operator` CLI handles host-side orchestration only.

The `journeyloop` CLI is baked into `companion-sandbox:latest`. When the API or IPC interface changes, rebuild the image.
