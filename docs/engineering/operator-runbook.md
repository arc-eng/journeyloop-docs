---
title: Companion Operator Infrastructure
description: How the companion operator VM is managed — heartbeat broadcasting, promotion pipeline, skill distribution, and cross-agent coordination.
---

# :material-server-network: Companion Operator Infrastructure

How the GCP companion operator is managed and how its agents coordinate. This covers the operational layer on the VM — not the companion agents themselves, but the infrastructure that runs and maintains them.

For provisioning and architecture, see the [Operator Guide](../companion/operator.md).

---

## Heartbeat Broadcasting

The operator can trigger heartbeats for all companion agents at once using `heartbeat-all.sh`:

```bash
cd ~/journeyloop/operator
./scripts/heartbeat-all.sh
```

This also uses an `--agent ALL` flag on `tasks.py`, which applies a command across every agent's queue simultaneously. It's used when the operator needs to push a coordinated update — for example, triggering all agents to pick up a configuration change or run a specific task.

Individual agent heartbeats are still managed by their own cron schedules. The broadcast mechanism is for operator-initiated coordination, not routine operation.

### Operator-Controlled Heartbeat Cron

The operator maintains a cron schedule for heartbeats with logging. This means:

- The operator (not individual agents) owns the heartbeat schedule
- Heartbeat runs are logged centrally on the VM, not just inside each agent session
- The operator can temporarily suppress or modify heartbeats for maintenance without touching each agent's config

---

## Promotion Pipeline

The `promote.sh` script handles promoting a new companion configuration to all live agents. As of Feb 2026, promotion **auto-migrates all live agents** — every companion agent picks up the new configuration as part of the promote run.

Previously, migration was a separate step. The consolidation removes the risk of partially-migrated fleets: every promote is atomic across all agents.

```bash title="Promote to all live agents"
cd ~/journeyloop/operator
./scripts/promote.sh
```

!!! warning "Promotion is irreversible during the run"
    There's no partial rollback once promote starts migrating agents. Test in staging first.

---

## Skill Distribution: Central Skills Directory

Companion agents share a central `skills/` directory on the operator VM rather than maintaining per-agent skill copies.

**Before:** Each agent workspace had its own copy of shared skills. Updating a skill meant merging changes into every agent.

**After:** A single `skills/` directory at the operator level is referenced by all agents. Agent workspaces contain only agent-specific files (`SOUL.md`, `MEMORY.md`, credentials, etc.).

This makes skill updates atomic — change once, all agents see it on next heartbeat. It also keeps agent workspaces lean: each workspace is only what's unique to that coach's agent.

The `operator/agent` files (templates, workspace structure) are kept separate from the shared `skills/` directory. Operator-level config doesn't bleed into agent workspaces.

---

## Task Queue for the Operator Agent

The main operator agent now has access to the task queue skill for cross-agent dispatch. The operator can add tasks to individual companion agent queues:

```bash
python3 skills/tasks/scripts/tasks.py add \
  --agent <coach-agent-id> \
  --title "Reload client profile for next session" \
  --added-by operator
```

This closes the coordination loop: the operator can not only broadcast heartbeats but also assign specific work to specific agents without sending a chat message.

Combined with `--agent ALL`, the operator can add the same task to every agent's queue at once.
