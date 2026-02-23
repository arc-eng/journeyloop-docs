---
title: Troubleshooting
description: Symptom → diagnosis → fix. The go-to guide when something breaks.
icon: material/wrench
---

# :material-wrench: Troubleshooting

**Structure:** symptom → diagnosis → fix. Every section has real commands. No fluff.

---

## 1. Quick Health Check

Run this first. Always.

```bash
# Is the gateway running?
openclaw gateway status

# Recent errors?
openclaw logs | tail -50

# Active sessions
openclaw sessions list

# Cron state
openclaw cron list

# Pi resources
free -h && df -h && uptime
```

| Check | Healthy | Unhealthy |
|---|---|---|
| Gateway status | `running` | `stopped` / no output |
| Log tail | No `ERROR` / `FATAL` lines | Stack traces, repeated crashes |
| Sessions | Expected agents listed | Missing agents, stuck `active` sessions |
| Cron | `enabled: true`, recent `lastRun` | `enabled: false`, `lastError` set |
| Memory (`free -h`) | >200MB free | <100MB free — OOM risk |
| Disk (`df -h`) | <80% on `/` | >90% — clean up logs/transcripts |

!!! tip "Keep a terminal alias"
    Add to `~/.bashrc`: `alias ochealth='openclaw gateway status && openclaw logs | tail -20 && free -h && df -h'`

---

## 2. Gateway Issues

### Gateway won't start

**Symptoms:** `openclaw gateway start` returns immediately, status stays `stopped`.

**Diagnose:**

```bash
# Run in foreground to see the actual error
openclaw gateway start --foreground

# Check for JSON syntax errors in config
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json'))" && echo "JSON OK"

# Check if port is already in use (default: 5050)
lsof -i :5050
# or
ss -tlnp | grep 5050
```

**Common causes & fixes:**

| Cause | Fix |
|---|---|
| Bad JSON in `openclaw.json` | Fix the syntax error reported by `python3` above |
| Port conflict | Kill the conflicting process: `kill $(lsof -t -i :5050)` |
| Missing env var | Check for required vars in config; set via `export` or `.env` file |
| Corrupt gateway store | `rm -rf ~/.openclaw/store && openclaw gateway start` (loses cron state — re-register crons) |

```bash
# Kill whatever is on port 5050 and restart
kill $(lsof -t -i :5050) && openclaw gateway start
```

---

### Gateway crashes / keeps restarting

**Symptoms:** Gateway starts but dies within minutes; repeated restarts in logs.

**Diagnose:**

```bash
# Check for OOM kills
dmesg | grep -i oom | tail -10

# Check gateway logs for last error before crash
openclaw logs | grep -E "FATAL|CRASH|killed" | tail -20

# Check memory at time of crash
dmesg | grep -E "oom|memory" | tail -20
```

**Fixes:**

=== "OOM (most common on Pi)"
    ```bash
    # Confirm OOM kill
    dmesg | grep -i oom | tail -5

    # Free memory: stop heavy background processes
    # Then restart gateway with fewer concurrent agents if needed
    openclaw gateway restart
    ```

=== "Crash loop"
    ```bash
    # Stop, wait, start clean
    openclaw gateway stop
    sleep 3
    openclaw gateway start

    # If still crashing, check config validity first
    python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json'))" && echo "OK"
    ```

---

### Config changes not taking effect

!!! info "What hot-reloads vs. what requires restart"
    | Setting | Hot reload | Restart required |
    |---|---|---|
    | Agent prompts / instructions | ✅ Yes | — |
    | Cron job schedule | ✅ Yes | — |
    | New agent registration | ❌ No | `openclaw gateway restart` |
    | Port / listener config | ❌ No | `openclaw gateway restart` |
    | Channel credentials (tokens) | ❌ No | `openclaw gateway restart` |

```bash
# For anything that needs a restart:
openclaw gateway restart

# Verify new config loaded (check startup logs)
openclaw logs | grep "config" | tail -10
```

---

## 3. Agent Not Responding

### Agent silent on Slack

**Diagnose:**

```bash
# Is it skipping due to mention requirement?
openclaw logs | grep "no-mention" | tail -20

# Is the agent registered?
openclaw agents list | grep <agent-name>

# Is there an active session stuck open?
openclaw sessions list | grep <agent-id>
```

**Fix by cause:**

| Symptom | Cause | Fix |
|---|---|---|
| Logs show `no-mention` | `requireMention: true`, message didn't @ the bot | @ the bot, or set `requireMention: false` |
| No log entries at all | Agent not registered / gateway not running | Re-register agent, restart gateway |
| Session shows `active` but no reply | Stuck session | Kill it: `openclaw sessions kill <session-id>` |

---

### Agent silent on Telegram

**Diagnose:**

```bash
# Check webhook status via Telegram Bot API
curl -s "https://api.telegram.org/bot<TOKEN>/getWebhookInfo" | python3 -m json.tool

# Check logs for Telegram errors
openclaw logs | grep -i "telegram\|webhook" | tail -20
```

**Fixes:**

```bash
# Re-set the webhook (replace TOKEN and URL)
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -d "url=https://<your-gateway-url>/telegram/<TOKEN>"

# Verify it took
curl -s "https://api.telegram.org/bot<TOKEN>/getWebhookInfo" | python3 -m json.tool
```

---

### Agent session stuck (never clears)

**Symptoms:** `openclaw sessions list` shows sessions in `active` state that have been running for hours.

**Diagnose:**

```bash
# Find stuck sessions
openclaw sessions list

# Check logs for sessions that never got a completion event
openclaw logs | grep "totalActive" | tail -30
```

**Fix:**

```bash
# Kill a specific stuck session
openclaw sessions kill <session-id>

# Nuclear: kill all active sessions for an agent
openclaw sessions list | grep <agent-id>
# then kill each one
```

---

### Agent lost identity / amnesia

**Symptoms:** Agent doesn't know its name, role, or backstory; behaves like a generic assistant.

**Diagnose:**

```bash
# Run the health check script
python3 scripts/agent_health_check.py

# Manual checks
openclaw agents list | grep <agent-id>   # is workspace/agentDir set?
ls ~/.openclaw/agents/<agent-id>/        # does SOUL.md exist?
cat ~/.openclaw/agents/<agent-id>/SOUL.md
```

!!! warning
    If `agentDir` is missing from the agent registration, every session reset means the agent loses its identity files entirely. Fix the registration config, not just the files.

**Fix:**

```bash
# 1. Verify registration has agentDir
openclaw agents show <agent-id>

# 2. If missing, edit openclaw.json to add workspace + agentDir, then restart
openclaw gateway restart

# 3. Confirm SOUL.md is in place
ls ~/.openclaw/agents/<agent-id>/SOUL.md
```

---

## 4. Slack-Specific Issues

See also: [Slack Agent Communication](slack-agent-communication.md) for full architecture.

### All bots respond to every message

**Cause:** `requireMention: false` on multiple agents in the same workspace.

**Diagnose:**

```bash
openclaw agents list  # check requireMention for each Slack agent
```

**Fix:**

Set `requireMention: true` on all agents that shouldn't respond to every message. Only the "default responder" agent (if any) should have it false.

```json
// openclaw.json — per-agent Slack config
{
  "requireMention": true
}
```

Then restart: `openclaw gateway restart`

---

### Thread replies ignored (bug #24816)

!!! warning "Known bug: openclaw/openclaw#24816"
    Thread replies in Slack carry an **implicit mention** — the platform treats them as directed at the bot even without an explicit `@mention`. The gateway does not always recognize this implicit mention, causing the agent to skip thread replies.

**Workaround:**

- Always use an explicit `@BotName` in thread replies when you need a response.
- Or set `requireMention: false` for agents that must respond in threads (accept the tradeoff of responding to all channel messages).

---

### Bot responds in DM instead of thread

**Cause:** Agent is using the `message` tool to send a reply instead of using the natural reply mechanism. The `message` tool sends a new message to a channel/DM rather than replying in-thread.

**Fix:** Update the agent's instructions to use natural reply (just respond) for Slack messages. The `message` tool should only be used for proactive/outbound messages, not responses.

---

### Slack socket disconnected

**Diagnose:**

```bash
openclaw logs | grep -i "socket\|reconnect\|disconnect" | tail -20
```

**Fix:**

```bash
# Gateway restart re-establishes all socket connections
openclaw gateway restart

# Verify socket connected after restart
openclaw logs | grep -i "socket\|connected" | tail -10
```

!!! tip
    If disconnects happen frequently, check Pi network stability and whether the Pi is sleeping/throttling. `uptime` and `dmesg | tail -20` can show intermittent issues.

---

## 5. Session & Memory Issues

### Agent has no context / amnesia

| Check | Command |
|---|---|
| Agent has `agentDir` set | `openclaw agents show <id>` |
| SOUL.md exists | `ls ~/.openclaw/agents/<id>/SOUL.md` |
| agentDir path is correct | `cat ~/.openclaw/agents/<id>/SOUL.md` |
| Session transcript exists | `ls ~/.openclaw/agents/<id>/sessions/` |

**Fix:**

```bash
# Add workspace + agentDir to agent config in openclaw.json
# Then restart
openclaw gateway restart
```

---

### Session not resetting when it should

**Diagnose:**

```bash
# Check resetByType config for the agent
openclaw agents show <agent-id> | grep -A5 "reset"
```

**Common `resetByType` values:**

| Value | Behavior |
|---|---|
| `"idle"` | Resets after idle timeout (default 24h for threads) |
| `"always"` | New session every message |
| `"never"` | Single persistent session |
| `"thread"` | One session per Slack thread |

Fix by updating the agent's `resetByType` in `openclaw.json` and restarting.

---

### Thread session expired unexpectedly

**Cause:** 24-hour idle timeout. The thread went quiet for >24h, session was cleared.

**Fix:** Just `@mention` the agent in the thread again — a new session starts. The agent won't have memory of the previous thread unless session transcripts are loaded in the prompt.

---

## 6. Cron Jobs

### Job not firing

**Diagnose:**

```bash
# List all cron jobs with status
openclaw cron list

# Look for: enabled, nextRun, lastRun, lastError
openclaw cron list | python3 -m json.tool
```

| Field to check | Expected | Problem |
|---|---|---|
| `enabled` | `true` | `false` → job is paused |
| `nextRun` | Near-future timestamp | Far future or null → schedule issue |
| `lastRun` | Recent timestamp | Never ran → first-time config issue |
| `lastError` | null / empty | Has value → job is erroring out |

**Fix:**

```bash
# Enable a disabled job
openclaw cron enable <job-id>

# Trigger a job manually to test
openclaw cron run <job-id>

# Check logs after manual run
openclaw logs | grep <job-id> | tail -20
```

---

### Job erroring

**Diagnose:**

```bash
# Get last error
openclaw cron list | python3 -c "
import json, sys
jobs = json.load(sys.stdin)
for j in jobs:
    if j.get('lastError'):
        print(j['id'], ':', j['lastError'])
"
```

**Common causes:**

| Error | Cause | Fix |
|---|---|---|
| `tool not found` | Agent missing a required tool | Add tool to agent config |
| `timeout` | Job taking longer than allowed | Increase timeout in cron config |
| `API error` / `401` | Expired token | Refresh token in `credentials/` |
| `ENOENT` / file missing | Script path wrong | Check `command` field in cron config |

---

### Jobs lost after restart

!!! info
    Cron state persists in the gateway store (`~/.openclaw/store`). Jobs are **not** lost on normal restart.

If jobs disappeared:
```bash
# Check if store was wiped
ls -la ~/.openclaw/store/

# Check gateway logs at startup for cron init
openclaw logs | grep -i "cron" | head -20
```

If the store was wiped (or you wiped it manually), cron jobs need to be re-registered.

---

## 7. Pi Resource Issues

### OOM kills

**Symptom:** Processes or the gateway die unexpectedly, especially during heavy use.

```bash
# Confirm OOM kills
dmesg | grep -i oom | tail -10

# Current memory state
free -h

# What's eating memory?
ps aux --sort=-%mem | head -15
```

!!! danger "Pi memory is limited"
    The Pi 4 has 4–8GB RAM but multiple concurrent agent runs + `gh` CLI + Node.js processes can hit limits fast. If you're running >3 agents concurrently, watch memory closely.

**Fix:**

```bash
# Kill memory hogs
kill <pid>

# Check if gh CLI is orphaned
ps aux | grep gh
kill $(pgrep -f "gh ")

# Restart gateway after clearing
openclaw gateway restart
```

---

### Disk full

```bash
# Check disk usage
df -h

# Find what's eating space
du -sh ~/.openclaw/agents/*/sessions/ | sort -h | tail -10
du -sh /tmp/openclaw/ 2>/dev/null

# Clean old session transcripts (keep last 30 days)
find ~/.openclaw/agents -name "*.jsonl" -mtime +30 -delete

# Clean old logs
find /tmp/openclaw/ -name "*.log" -mtime +7 -delete
```

!!! warning "Don't delete active session files"
    Only clean sessions older than your retention window. Active sessions are in `sessions/` with recent timestamps.

---

### High CPU

```bash
# Snapshot of top processes
top -b -n1 | head -20

# Specifically check for Node (gateway) and Python (agent scripts)
ps aux --sort=-%cpu | grep -E "node|python" | head -10
```

**Cause:** Usually multiple agent runs in parallel. Each active session with a model call holds CPU.

**Fix:**

```bash
# Kill stuck sessions to free CPU
openclaw sessions list
openclaw sessions kill <session-id>

# Or restart gateway to clear everything
openclaw gateway restart
```

---

## 8. Common Config Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Missing `workspace` on agent | Agent can't find files, may error on tool calls | Add `"workspace": "/home/dobby/.openclaw/workspace"` to agent config |
| Missing `agentDir` on agent | Amnesia on session reset | Add `"agentDir": "agents/<name>"` to agent config |
| `replyToMode: "off"` | Agent replies in new messages, not threads (Slack) | Set `"replyToMode": "thread"` |
| Wrong calendar ID | Events not appearing in agent responses | Get correct ID from Google Calendar settings → copy the calendar ID |
| Expired OAuth token | API calls fail with 401/403 | Refresh token in `~/.openclaw/credentials/` |
| `requireMention: false` on multiple agents | All bots respond to everything | Set `requireMention: true` on agents that should only respond when @-mentioned |
| Bad cron schedule expression | Job never fires or fires constantly | Validate cron expression at [crontab.guru](https://crontab.guru) |

```bash
# After any config change to openclaw.json, validate JSON first:
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json'))" && echo "JSON valid"

# Then restart
openclaw gateway restart
```

---

## 9. Log Locations & How to Read Them

### File locations

| Log type | Location |
|---|---|
| Gateway logs | `/tmp/openclaw/openclaw-YYYY-MM-DD.log` |
| Agent session transcripts | `~/.openclaw/agents/<id>/sessions/*.jsonl` |
| Cron logs | Inside gateway logs — grep for `"cron"` |

```bash
# Today's gateway log
cat /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -100

# All logs, most recent first
ls -lt /tmp/openclaw/*.log

# Grep for errors across all logs
grep -h "ERROR\|FATAL" /tmp/openclaw/*.log | tail -30

# Grep for a specific agent
grep -h "<agent-id>" /tmp/openclaw/*.log | tail -30
```

---

### Parsing JSON logs

Gateway logs are newline-delimited JSON. Use Python to extract fields:

```bash
# Extract just the message field from recent errors
tail -200 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        obj = json.loads(line)
        if obj.get('level') in ('error', 'fatal'):
            print(obj.get('time', ''), obj.get('msg', ''), obj.get('error', ''))
    except:
        pass
"

# Show all fields for lines matching a pattern
grep "session" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        print(json.dumps(json.loads(line), indent=2))
    except:
        print(line, end='')
" | head -100
```

---

### Reading session transcripts

```bash
# List sessions for an agent
ls -lt ~/.openclaw/agents/<agent-id>/sessions/

# Read a session (each line is a JSON turn)
cat ~/.openclaw/agents/<agent-id>/sessions/<session-id>.jsonl | python3 -c "
import sys, json
for line in sys.stdin:
    obj = json.loads(line)
    role = obj.get('role', '?')
    content = obj.get('content', '')
    if isinstance(content, list):
        content = ' '.join(c.get('text', '') for c in content if isinstance(c, dict))
    print(f'[{role}] {str(content)[:200]}')
"
```

---

## 10. Emergency Recovery

### Full gateway reset

```bash
openclaw gateway stop && sleep 2 && openclaw gateway start

# Verify it's up
openclaw gateway status
openclaw logs | tail -20
```

---

### Config backup & rollback

!!! danger "Always back up before editing `openclaw.json`"
    ```bash
    cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d-%H%M%S)
    ```

```bash
# Back up current config
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d-%H%M%S)

# List backups
ls -lt ~/.openclaw/openclaw.json.bak.*

# Roll back to a specific backup
cp ~/.openclaw/openclaw.json.bak.20260223-140000 ~/.openclaw/openclaw.json

# Validate before restarting
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json'))" && echo "JSON valid"

# Restart
openclaw gateway restart
```

---

### Gateway store reset (last resort)

!!! danger "This wipes cron state and persisted session data"
    Only do this if the gateway refuses to start and you've ruled out config/port issues.

```bash
# Stop gateway first
openclaw gateway stop

# Back up the store
cp -r ~/.openclaw/store ~/.openclaw/store.bak.$(date +%Y%m%d-%H%M%S)

# Wipe the store
rm -rf ~/.openclaw/store

# Start fresh
openclaw gateway start

# Re-register any cron jobs that were lost
```

---

### Quick reference: most common fixes

| Problem | One-liner fix |
|---|---|
| Gateway won't start — port conflict | `kill $(lsof -t -i :5050) && openclaw gateway start` |
| Config change not reflected | `openclaw gateway restart` |
| Stuck session | `openclaw sessions kill <id>` |
| Agent amnesia | Check `agentDir` in config, `openclaw gateway restart` |
| Cron job disabled | `openclaw cron enable <id>` |
| Socket disconnected | `openclaw gateway restart` |
| Disk full | `find ~/.openclaw/agents -name "*.jsonl" -mtime +30 -delete` |
| OOM crash | `openclaw gateway restart` + reduce concurrent agents |
| Roll back config | `cp ~/.openclaw/openclaw.json.bak.<ts> ~/.openclaw/openclaw.json && openclaw gateway restart` |
