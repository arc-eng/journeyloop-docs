# JourneyLoop Docs

Internal documentation for the JourneyLoop platform — architecture decisions, infrastructure setup, engineering process, and agent protocols.

This is not code documentation. It captures **why things are built the way they are** and **how to operate them**.

---

## Platform

Core concepts, data model, and the SHIFT coaching framework.

- [Platform Overview](platform/overview.md) — what JourneyLoop is, the data model (coaches, clients, sessions, goals), and how sessions flow through the system
- [SHIFT Framework](platform/overview.md#the-shift-framework) — the 5-principle coaching quality model used for AI session analysis

---

## Companion

The AI companion feature — how it's architected, provisioned, and operated.

- [Operator Guide](companion/operator.md) — architecture (OpenClaw on GCP), provisioning flow, key design decisions, operations runbook

---

## Engineering

How the team works and how decisions are made.

- [Process](engineering/process.md) — issue lifecycle, agent roles, planning conventions

---

*Last updated: 2026-02-23 — docs are maintained by the Docs Agent. To add or update a page, send a `WRITE` message to `agent:docs:main`.*
