---
title: JourneyLoop Docs
description: Internal documentation for the JourneyLoop platform — architecture, infrastructure, and engineering process.
---

# :material-book-open-variant: JourneyLoop Docs

Internal documentation for the JourneyLoop platform — architecture decisions, infrastructure setup, engineering process, and agent protocols.

!!! info "What this is"
    This is **not** code documentation. It captures **why things are built the way they are** and **how to operate them**.

---

<div class="grid cards" markdown>

-   :material-layers-outline: **Platform**

    ---

    Core concepts, the data model, and the SHIFT coaching quality framework.

    - [Platform Overview](platform/overview.md) — what JourneyLoop is, the data model (coaches, clients, sessions, goals), and how sessions flow through the system
    - [SHIFT Framework](platform/overview.md#the-shift-framework) — the 5-principle coaching quality model used for AI session analysis

    [:octicons-arrow-right-24: Explore Platform](platform/index.md)

-   :material-robot-outline: **Companion**

    ---

    The AI companion feature — how it's architected, provisioned, and operated.

    - [Operator Guide](companion/operator.md) — architecture (OpenClaw on GCP), provisioning flow, key design decisions, operations runbook

    [:octicons-arrow-right-24: Explore Companion](companion/index.md)

-   :material-cog-outline: **Engineering**

    ---

    How the team works and how decisions are made.

    - [Process](engineering/process.md) — issue lifecycle, agent roles, planning conventions
    - [Agent Roster](engineering/agent-roster.md) — the full AI agent team, session keys, responsibilities
    - [Slack Agent Communication](engineering/slack-agent-communication.md) — thread-per-session model, gateway config, agent behavior, known limitations
    - [Troubleshooting](engineering/troubleshooting.md) — symptom → diagnosis → fix for gateway, agents, sessions, cron, and Pi resources

    [:octicons-arrow-right-24: Explore Engineering](engineering/index.md)

</div>

---

*:material-clock-edit-outline: Last updated: 2026-02-23 — docs are maintained by the Docs Agent. To add or update a page, send a `WRITE` message to `agent:docs:main`.*
