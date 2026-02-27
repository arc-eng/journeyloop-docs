---
title: Agent Task Queue
description: How agents delegate and coordinate work using the persistent task queue skill.
---

# :material-format-list-checks: Agent Task Queue

The task queue is how JourneyLoop agents assign work to each other and track it across heartbeat cycles. It replaces ad-hoc Slack messages and ephemeral memory as the coordination layer for async agent-to-agent delegation.

---

## Why It Exists

Before the task queue, agents had no reliable way to assign work to another agent. A PM could send a Slack message to the SWE agent, but if the SWE session restarted — or if the message was buried — the task was lost. Agents also couldn't inspect what work was pending for their teammates.

The task queue solves this with three properties:

- **Persistent** — tasks survive session restarts and gateway reboots
- **Agent-scoped** — each agent has its own queue; no shared mutable state
- **Inspectable** — JSON files in the workspace, readable by any agent or human

---

## Architecture

Each agent's queue lives at:

```
~/.openclaw/workspace/agents/<name>/TASKS.json
```

The file is created automatically the first time a task is added to an agent. All agents can read and write all queues — there's no access control. Trust is the protocol.

Tasks have five statuses: `pending` → `in-progress` → `done` / `failed` / `dropped`.

**Priority** is `high` > `normal` > `low`. Within the same priority tier, the oldest task is picked first (FIFO). This keeps the queue predictable and prevents starvation.

---

## Heartbeat Integration

On every heartbeat, each agent:

1. Picks the next pending task: `python3 skills/tasks/scripts/tasks.py pick --agent <name>`
2. If nothing is returned — stops and replies `HEARTBEAT_OK`
3. Works on the task
4. Marks it done or failed

**One task per heartbeat. No exceptions.** Agents don't loop through the queue in a single run. This keeps each heartbeat bounded and ensures that long tasks don't block the agent indefinitely.

!!! warning "Don't chain tasks in a single heartbeat"
    Picking a second task after finishing the first leads to unbounded run times and makes failure harder to diagnose. The next heartbeat will pick the next task.

---

## Cross-Agent Assignment

Any agent can add a task for any other agent:

```bash
python3 skills/tasks/scripts/tasks.py add \
  --agent swe \
  --title "Implement companion memory model" \
  --notes "See tech-spec: planning/companion/memory-system/tech-spec.md" \
  --priority high \
  --added-by cto
```

This is the primary coordination mechanism between agents. When the PM finishes a brief, it adds a task to the UX queue. When the CTO finishes a tech spec, it adds a task to the SWE queue. No Slack ping required.

---

## Failure Handling

Failed tasks are **never automatically cleaned up**. They stay in the queue with `status: "failed"`, the error message, and a retry count.

This is intentional. Failed tasks are diagnostic signals — they tell the `doctor` agent (or Dobby) what broke, when, and how many times it's been tried. Cleaning them up automatically would erase that history.

```mermaid
stateDiagram-v2
    [*] --> pending : task added
    pending --> in_progress : pick
    in_progress --> done : done
    in_progress --> failed : fail
    in_progress --> dropped : drop
    failed --> in_progress : retry (manual)
    failed --> dropped : drop (doctor decision)
```

Done and dropped tasks older than 7 days are removed by `clean`. Failed tasks are exempt.

---

## CLI Quick Reference

The skill lives at `skills/tasks/scripts/tasks.py`. Full reference in `skills/tasks/SKILL.md`.

| Command | Purpose |
|---------|---------|
| `pick --agent NAME` | Pick next pending task (marks in-progress) |
| `done ID --agent NAME --result "..."` | Mark done with summary |
| `fail ID --agent NAME --error "..."` | Mark failed — persists for diagnosis |
| `add --agent NAME --title "..."` | Add a task to any agent's queue |
| `list [--agent NAME] [--status pending]` | Inspect queues |
| `digest [--since TIMESTAMP]` | Done/failed summary for CEO digest |
| `clean [--days 7]` | Remove old done/dropped tasks |

Task IDs support prefix matching — the first 8 characters are enough.
