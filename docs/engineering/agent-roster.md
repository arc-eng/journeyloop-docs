# Agent Roster & Org Structure

This page documents the AI agent team that builds and operates JourneyLoop, how they're organized, and how they collaborate.

---

## Overview

JourneyLoop is built and operated by a team of AI agents running on OpenClaw. Each agent has a defined role, a set of responsibilities, and a communication protocol. Marco Lamina (founder) is the human stakeholder who approves work, makes product decisions, and owns escalation.

Agents communicate primarily via:

- **Slack** — human-facing status updates and reviews (Marco reads these)
- **`sessions_send`** — agent-to-agent task dispatch (direct session messaging, no Slack @mentions)
- **GitHub issues and PRs** — structured work artifacts

---

## The JourneyLoop Agent Team

### CTO (Chief Technology Officer)

**Session key:** `agent:cto:main`
**Slack:** posts to `#development`, `#reviews`

**Role:** Owns technical architecture for JourneyLoop. Translates product specs into engineering plans and ensures the codebase stays healthy.

**Responsibilities:**

- Write tech specs (`planning/<feature>/tech-spec.md`) for issues labeled `needs-tech-spec`
- Author Architecture Decision Records (ADRs) as GitHub issues labeled `adr`
- Review PRs for architectural consistency, security, and code quality
- Identify and quantify technical debt
- Maintain architectural consistency across `arc-eng/journeyloop` and `arc-eng/journeyloop-startup-assistant`

**Key constraint:** Never auto-proceeds — always checks in with Marco via `#reviews` before starting a tech spec. Iterates in conversation with Marco rather than writing complete specs upfront.

**Signs GitHub comments:** `— CTO Agent`

---

### PM (Product Owner)

**Session key:** `agent:pm:main`
**Slack:** posts to `#development`, `#reviews`

**Role:** Owns the product backlog for JourneyLoop. Writes specs, manages issue lifecycle, and keeps the team aligned on priorities.

**Responsibilities:**

- Write user stories, feature specs, and briefs
- Manage GitHub issue labels through the lifecycle
- Prioritize the backlog in consultation with Marco
- Escalate when product decisions require founder input

**Does not:** Make architecture decisions, manage sprints, or touch code.

---

### UX Engineer

**Session key:** `agent:ux:main`
**Slack:** posts to `#development`, `#reviews`

**Role:** Owns design principles, wireframes, and user experience for JourneyLoop.

**Responsibilities:**

- Design UI layouts and wireframes (HTML/CSS prototypes)
- Define and maintain design principles and component patterns
- Review PRs for UX quality
- Champion the user's perspective in technical discussions
- Write `ux.md` files in feature planning folders

**Triggered by:** Issues labeled `needs-ux` (dispatched by PM).

---

### SWE (Senior Software Engineer)

**Session key:** `agent:swe:main`
**Slack:** posts to `#development`

**Role:** Implements features, fixes bugs, writes tests, and maintains code quality across `arc-eng/journeyloop`.

**Responsibilities:**

- Implement features from tech specs when issues reach `ready-for-dev`
- Follow gitflow: feature branch (named with issue number) → commits → PR
- Write and maintain tests
- Maintain `CLAUDE.md` files across repos
- Fix bugs and handle tech debt items

**Note:** SWE is not auto-triggered. `ready-for-dev` is the final pipeline stage — Marco initiates dev work manually.

---

### Docs Agent

**Session key:** `agent:docs:main`

**Role:** Owns the JourneyLoop documentation site (MkDocs Material, live at http://192.168.1.30:8001).

**Responsibilities:**

- Write and update markdown documentation files on instruction
- Maintain `mkdocs.yml` navigation
- Rebuild and restart the MkDocs service after every write
- Answer retrieval queries with doc content

Not conversational. Accepts structured messages from other agents:

| Message | Purpose |
|---|---|
| `WRITE: <path> | <title> | <content>` | Write or update a doc |
| `READ: <query>` | Retrieve relevant docs |
| `LIST` | List all current docs |

**Repo:** `arc-eng/journeyloop-docs` at `/home/dobby/.openclaw/workspace/repos/journeyloop-docs`

---

### Ray (Practitioner Advisor & Product Tester)

**Session key:** `agent:ray:main`

**Role:** A practicing executive coach who uses JourneyLoop on staging. Ray serves as both a real user and a coaching practitioner advisor.

**As product tester:** Logs into staging.journeyloop.ai and uses the product as a real coach — not running test scripts, but doing actual coaching work and surfacing friction.

**As practitioner advisor:** Provides gut-checks on coach workflow design, UX assumptions, and feature trade-offs from a practitioner's perspective.

**Active clients (staging):** David Chen, Neha Sharma, Samantha Lin

**Reach:** Other agents dispatch tasks to Ray via `sessions_send`. Ray responds conversationally — give context, ask something specific.

---

### Audit Agent

**Session key:** `agent:audit:main`

**Role:** Monitors all agent activity across the OpenClaw system and maintains a structured audit trail.

**Runs:** Every 20 minutes (cron-based heartbeat).

**Output:** Daily log files at `logs/YYYY-MM-DD-<slug>.md` in the audit agent workspace. Flags errors, unexpected behavior, or security issues when found.

---

### Doctor

**Session key:** `agent:doctor:main`

**Role:** Infrastructure diagnostic tool. Audits the multi-agent setup for misconfigurations, isolation failures, and policy violations.

**Style:** Clinical, precise — no filler. Reports findings as Critical / Warning / Passing with suggested fixes.

---

## Supporting Agents (non-JourneyLoop)

These agents run on the same OpenClaw instance but serve Dobby's personal or other-client contexts:

| Agent | Purpose |
|---|---|
| `coding-agent` | General-purpose coding assistant for Dobby |
| `screen-dev` | Frontend development for Dobby's 5" touchscreen UI (Flask + Alpine.js + Tailwind, Raspberry Pi 5) |
| `secops` | Security auditing for OpenClaw deployments and host hardening |
| `business-analyst` | Daily strategic business research and analysis for JourneyLoop |
| `caitlin-assistant` | Business assistant for Caitlin Ryan (ocean-themed wellness pivot) |
| `voice` | Reformats text for voice/TTS output |
| `coach-test` (Sarah Chen) | Staging coaching companion for a test coach persona |

---

## Collaboration Protocols

### Issue Lifecycle

Issues in `arc-eng/journeyloop` move through these labels:

```
needs-spec → needs-ux → needs-tech-spec → needs-review → ready-for-dev → in-progress → done
```

Each label triggers the responsible agent. Marco approves transitions at key gates (`needs-review`).

### Agent-to-Agent Communication

Agents dispatch work to each other via `sessions_send(sessionKey="agent:<name>:main", ...)`. **Never via Slack @mentions** — bots don't receive those reliably.

### Slack Channels

| Channel | Purpose |
|---|---|
| `#development` | Work log — short updates after every completed task |
| `#reviews` | Escalation to Marco — always includes specific questions or decisions needed |
| `#bot-problems` | Blockers and errors that need human attention |

### Marco's Role

Marco Lamina is the founder and primary stakeholder. He:

- Approves work before agents start (no autonomous sprinting)
- Makes product prioritization calls
- Reviews tech specs and UX before dev proceeds
- Has final say on architecture trade-offs, vendor decisions, and scope

Agents escalate to Marco when: architecture has significant trade-offs, there are technical blockers requiring business context, or effort estimate exceeds expectations by >2x.

---

## Repos

| Repo | Purpose |
|---|---|
| `arc-eng/journeyloop` | Main Django codebase — user stories, feature specs, bugs, tech specs, ADRs |
| `arc-eng/journeyloop-startup-assistant` | Planning — roadmap, business intel, revenue analysis, strategic decisions |
| `arc-eng/journeyloop-docs` | Documentation site (this site) |
| `arc-eng/companion-operator` | OpenClaw companion infrastructure scripts |

---

*Last updated: 2026-02-23*
