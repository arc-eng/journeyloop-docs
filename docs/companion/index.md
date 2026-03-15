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

-   :material-monitor-share: **Page Push**

    ---

    How the companion pushes UI pages to the coach during sessions:

    - Which pages can be pushed (client profiles, sessions, calendar, observations)
    - Push panel design — overlay with draggable divider, not layout shift
    - State persistence across reloads
    - Spotlight: companion-initiated highlights to direct coach attention

    [:octicons-arrow-right-24: Read the Page Push Guide](page-push.md)

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

-   :material-file-code-outline: **Dotfile IPC**

    ---

    How sandboxed agents request side-effects from the operator:

    - JSON `*.request.json` files for file delivery and Telegram UI
    - Auto-resolved delivery target from `.credentials.json`
    - `journeyloop` vs `jl-operator` CLI split
    - Monitor reliability design

    [:octicons-arrow-right-24: Dotfile IPC Reference](dotfile-ipc.md)

-   :material-chart-line: **Monitoring Stack**

    ---

    Prometheus + Grafana observability on the GCP VM:

    - Message flow, token cost, session health
    - Docker container + VM resource metrics
    - SSH tunnel access to Grafana
    - OpenClaw OTLP configuration

    [:octicons-arrow-right-24: Monitoring Stack](monitoring.md)

</div>
