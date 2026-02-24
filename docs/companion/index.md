---
title: Companion
description: Documentation for the AI companion feature — architecture, provisioning, and operations.
---

# :material-robot-outline: Companion

Documentation for the AI companion feature — how it's architected, deployed, and operated.

!!! info "What the companion is"
    The companion is a per-coach AI agent running on a dedicated GCP VM (the "companion operator"), proxied by the Django app. Each coach gets their own isolated agent with its own memory, workspace, and sandboxed CLI access to JourneyLoop data.

This section covers the **infrastructure and operations layer**: how agents are provisioned, how the system is wired together, and the key decisions behind the design.

---

<div class="grid cards" markdown>

-   :material-server-outline: **Operator Guide**

    ---

    Everything you need to understand how the companion works end-to-end:

    - Two-layer architecture (Django + OpenClaw)
    - How provisioning works
    - Why keys are operator-generated
    - Sandbox isolation configuration
    - GCP VM operations runbook

    Includes the key design decisions and their rationale.

    [:octicons-arrow-right-24: Read the Operator Guide](operator.md)

-   :material-school-outline: **Training Program**

    ---

    How companion quality improves over time:

    - Scenario-based evaluation loop
    - Four-dimension scoring rubric (Warm, Efficient, Error-free, Accessible)
    - Template refinement and the best-templates pipeline
    - What companions learn about coaches during bootstrap
    - Per-client relationship files
    - Production promotion — why changes never ship automatically

    [:octicons-arrow-right-24: Read the Training Program](training.md)

</div>
