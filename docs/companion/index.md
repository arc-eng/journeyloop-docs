# Companion

Documentation for the AI companion feature — how it's architected, deployed, and operated.

The companion is a per-coach AI agent running on a dedicated GCP VM (the "companion operator"), proxied by the Django app. Each coach gets their own isolated agent with its own memory, workspace, and sandboxed CLI access to JourneyLoop data.

This section covers the **infrastructure and operations layer**: how agents are provisioned, how the system is wired together, and the key decisions behind the design.

---

## What's in this section

**[Operator Guide](operator.md)**
Everything you need to understand how the companion works end-to-end: the two-layer architecture (Django + OpenClaw), how provisioning works, why keys are operator-generated, how sandbox isolation is configured, and how to operate the GCP VM. Includes the key design decisions and their rationale.
