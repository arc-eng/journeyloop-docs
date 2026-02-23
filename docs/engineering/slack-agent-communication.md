---
title: Slack Agent Communication Architecture
description: How JourneyLoop AI agents communicate over Slack — thread-per-session model, gateway config, agent behavior rules, and troubleshooting.
---

# :material-slack: Slack Agent Communication Architecture

How JourneyLoop AI agents communicate over Slack: the thread-per-session model, OpenClaw gateway configuration, expected agent behavior, session key mapping, known limitations, and troubleshooting.

!!! info "Internal reference"
    This doc is for the engineering team. It describes the configuration active on all five agent Slack accounts (PM, CTO, UX, SWE, Ray) as of **2026-02-23**.

---

## 1. Overview

### Thread-Per-Session Model

Every Slack channel thread is an **isolated agent session**. When you post to a channel and the agent replies, OpenClaw creates a thread automatically and binds a new session to it. Subsequent messages in that thread are part of the same session — the agent has context continuity for that entire conversation.

```
1. User posts "@SWE implement #126" in #development
2. OpenClaw routes to SWE → new channel session
3. SWE replies → Slack auto-creates a thread (replyToMode: "first")
4. Thread TS captured → new session key: agent:swe:slack:channel:<id>:thread:<ts>
5. User replies in thread with @SWE → routes to existing thread session
6. SWE continues with full thread context
```

### Why Thread-Per-Session?

| Concern | Before (flat channel) | After (thread-per-session) |
|---|---|---|
| Context isolation | All channel messages bleed into one session | Each thread = its own conversation |
| Parallel conversations | Agents confused by interleaved topics | Clean isolation per thread |
| History retrieval | Full channel history loaded (noisy) | Thread history only (`historyScope: "thread"`) |
| Auditability | Hard to trace a request end-to-end | Thread = single logical unit |

DMs and group DMs are intentionally **not** threaded — they stay as flat conversations (see [Gateway Configuration](#2-gateway-configuration) below).

---

## 2. Gateway Configuration

These settings are applied to **all five Slack accounts** in each agent's `openclaw.json`.

### `replyToMode`

```json
"replyToMode": "first"
```

The bot's **first reply** to a channel message creates a Slack thread. This is what triggers thread creation — the bot does not post in the channel root for subsequent replies.

### `replyToModeByChatType`

```json
"replyToModeByChatType": {
  "channel": "first",
  "direct": "off",
  "group": "off"
}
```

- **`channel: "first"`** — thread on first reply (channel posts only)
- **`direct: "off"`** — DMs remain flat; no thread wrapping
- **`group: "off"`** — group DMs remain flat

### Thread History

```json
"thread": {
  "historyScope": "thread",
  "inheritParent": false,
  "initialHistoryLimit": 20
}
```

`historyScope: "thread"`
:   Agent only sees messages within the thread. Parent channel messages are excluded.

`inheritParent: false`
:   Thread sessions do **not** inherit context from the parent channel session. Each thread starts fresh.

`initialHistoryLimit: 20`
:   On first load of an existing thread, up to 20 prior messages are loaded for context.

### Session Reset

```json
"session": {
  "resetByType": {
    "thread": {
      "mode": "idle",
      "idleMinutes": 1440
    }
  }
}
```

Thread sessions reset after **24 hours of inactivity** (`1440` minutes). After reset, the next message in the thread starts a fresh session — prior thread history is re-loaded up to `initialHistoryLimit`.

---

## 3. Agent Instructions

These rules apply to all JourneyLoop agents (PM, CTO, UX, SWE, Ray). They are codified in each agent's `AGENTS.md` and the shared `ORG.md` and `MEMORY.md`.

### Reply Behavior by Context

=== "Channel Thread"
    - Reply **naturally** — OpenClaw routes the reply back to the same thread automatically
    - Do **not** use the `message` tool for thread replies; that would create a separate DM or mis-routed post
    - Stay within the thread; never surface a thread response back to the channel root

=== "Direct Message"
    - Use the **`message` tool** explicitly to reply
    - DMs have no thread; responses go directly to the DM conversation
    - Session key is DM-scoped (see [Session Keys](#4-session-keys))

=== "Proactive / Agent-Initiated Post"
    - Always use the `message` tool with a fully-qualified `target` (channel ID or user ID)
    - Do not assume a channel context exists
    - If starting a new channel topic, post to the channel root — do not reply to an existing thread unless explicitly continuing one

### The Acknowledge-First Rule

!!! warning "Always acknowledge before starting long tasks"
    If a request will take more than a few seconds, **send a short acknowledgment first**, then execute.

    ```
    ✅ "On it — reviewing the spec now."
    ✅ "Got it, running the build."
    ❌ (silence for 30 seconds, then a wall of output)
    ```

    This prevents users from thinking the bot missed the message, and makes thread conversations feel responsive.

---

## 4. Session Keys

Session keys are how OpenClaw tracks conversation state. Each distinct context gets its own key.

| Context | Session Key Format |
|---|---|
| Channel (root) | `agent:<id>:slack:channel:<channelId>` |
| Channel Thread | `agent:<id>:slack:channel:<channelId>:thread:<threadTs>` |
| Direct Message | `agent:<id>:slack:dm:<userId>` |

**Key points:**

- `<id>` is the agent identifier (e.g. `pm`, `cto`, `ux`, `swe`, `ray`)
- `<channelId>` is the Slack channel ID (e.g. `C08ABCDEF`)
- `<threadTs>` is the Slack thread timestamp of the **parent message** (e.g. `1740349800.123456`)
- `<userId>` is the Slack user ID for DMs (e.g. `U01ABCDEF`)

A channel root message and a thread off that same message are **different sessions**. This is intentional — threads are isolated.

---

## 5. Known Limitations

### :material-bug: Implicit Mention Bug — `openclaw/openclaw#24816`

**Issue:** Thread follow-up messages that do **not** `@mention` the bot are silently ignored. The agent never sees them.

**Root cause:** OpenClaw's implicit mention check compares `parent_user_id === botUserId` — i.e., it only triggers on threads where the *bot started the thread*. Since the thread is created by the user's channel post (and the bot replies into it), `parent_user_id` is the user, not the bot. The implicit mention never fires.

**Status:** Filed at [openclaw/openclaw#24816](https://github.com/openclaw/openclaw/issues/24816). No fix timeline yet.

**Workaround:**

!!! tip "Always @mention the bot in thread replies"
    Every message in a thread directed at the agent must explicitly `@mention` the bot account.

    ```
    ❌ "Can you check the spec again?"          ← bot never sees this
    ✅ "@pm-agent Can you check the spec again?" ← works correctly
    ```

    This applies to **all** thread follow-ups, not just the first one. Document this in any user-facing onboarding for Slack.

---

## 6. Troubleshooting

### Agent not responding in a thread

**Symptoms:** You reply in a thread, no response.

| Check | What to do |
|---|---|
| Did you `@mention` the bot? | Required due to bug #24816 (see Known Limitations above). Add the `@mention` and retry. |
| Is the session idle-expired? | Sessions reset after 24h idle. The next `@mention` in the thread will restart the session with the last 20 messages as context. |
| Is the agent account active? | Check `openclaw gateway status` on Dobby. All five agent Slack accounts run through the same gateway. |
| Is the channel configured? | Confirm the channel is in the agent's allowed channels list in `openclaw.json`. |

### Response goes to DM instead of the thread

**Symptoms:** Agent responds in a DM to you rather than in the channel thread.

**Cause:** The agent used the `message` tool instead of a natural reply, and the tool resolved to your DM.

**Fix:** Update the agent's `AGENTS.md` — reinforce that thread replies must be natural replies, not `message` tool calls. If an individual agent instance did this, it may have hallucinated a tool call; a session reset usually clears it.


### Response goes to channel root instead of the thread

**Symptoms:** Agent's reply appears in the channel as a top-level post, not in the thread.

**Cause:** Usually a `message` tool call targeting the channel ID without a `threadTs`. Or the agent replied before OpenClaw bound the thread session.

**Fix:** Same as above — reinforce natural reply behavior in `AGENTS.md`. Check whether `replyToMode` is set to `"first"` in `openclaw.json`; if it's `"off"` or missing, threading is disabled.

### Agent has no context of earlier thread messages

**Symptoms:** Agent responds as if the thread is brand new, ignoring prior context.

**Cause 1:** Session idle-expired (24h) — the session was reset. Prior history is re-loaded up to `initialHistoryLimit: 20`.

**Cause 2:** `inheritParent: false` is correct behavior — threads intentionally don't inherit parent channel context.

**Cause 3:** `historyScope` misconfigured. Verify `"historyScope": "thread"` in the agent's gateway config.

### Two agents responding to the same thread

**Symptoms:** Both e.g. PM and CTO agents reply to the same thread message.

**Cause:** The message `@mentioned` both bots, or a channel is subscribed by multiple agents.

**Fix:** Be explicit about which agent you're addressing. Each agent monitors specific channels — review channel assignments in each agent's `openclaw.json`.

---

*:material-clock-edit-outline: Last updated: 2026-02-23 — configuration applied in the Slack thread sessions restructure.*
