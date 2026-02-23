# Companion Operator Guide

How the companion is provisioned, architected, and deployed within the JourneyLoop platform.

---

## Architecture Overview

The companion is a **Django app** (`companion/`) inside the existing JourneyLoop project. It shares the database, auth system, and infrastructure with the existing portal — but its AI layer is fully greenfield.

```
journeyloop/
  coaching/              # Existing — session recording, client management
  calendar_integration/  # Existing — Google Calendar, Recall.ai
  payments/              # Existing — Stripe subscriptions
  client_portal/         # Existing — client-facing views
  companion/             # NEW — companion app
    models/
      memory.py          # CompanionMemory, SessionIntelligence, ClientKnowledge
      skills.py          # CompanionSkill
      personality.py     # CompanionPersonality, CompanionState
      scheduling.py      # CompanionSchedule, ScheduleExecution
      context.py         # CompanionContext
      interactions.py    # MicroInteraction
    engine/
      prompt_assembler.py  # Personality + skills + memory → system prompt
      router.py            # LLM response → panel routing via SSE
      capabilities.py      # Capability gating (bootstrap state)
      personality.py       # Self-modification logic
      skills.py            # Skill loading, creation, retirement
    scheduling/
      engine.py          # APScheduler polling job
      hooks.py           # Session event hooks
      executor.py        # Scheduled action execution
      delivery.py        # Email / in-app delivery
    api/
      views.py           # REST endpoints
      events.py          # SSE push helpers
    templates/
      companion/
        page.html        # Main layout (golden ratio grid)
        partials/        # HTMX fragments for each panel
```

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Deployment | Same Django project, new app | Shared DB + auth. No API gateway complexity |
| AI layer | Fully greenfield | Existing prompts don't fit companion model. No reuse of AICoach/SessionAnalyzer |
| Frontend | Server-rendered HTMX + SSE | Already in stack. No SPA framework needed |
| Real-time | django-eventstream | Already in stack. One SSE connection per coach |
| Task queue | django-rq | Already in stack. Async message processing |
| Structured output | Pydantic-AI | Already in stack. LLM returns typed CompanionResponse |
| Token counting | tiktoken | Already in stack. Budget enforcement at save time |

## Provisioning Flow

### New Coach — What Happens at Signup

1. **Coach creates account** (existing portal auth)
2. **First visit to `/companion/`** triggers provisioning:
    - `provision_personality(coach)` — creates 6 empty personality layer rows
    - `CompanionState` created with `bootstrap_status='not_started'`
    - `CompanionContext` created with `mode='bootstrap'`
3. **Bootstrap begins** — companion's system prompt includes bootstrap instructions
4. **Through conversation**, companion fills soul, identity, and coach knowledge layers
5. **When all 3 MVP layers are set** → `capabilities_unlocked = True`, `bootstrap_status = 'complete'`
6. **Full capabilities unlock** — session prep, client analysis, scheduling, proactive nudges

### What Gets Created Per Coach

| Model | Count | Purpose |
|-------|-------|---------|
| `CompanionPersonality` | 6 rows | One per layer (soul, identity, coach_knowledge, instructions, environment, client_persona) |
| `CompanionState` | 1 row | Bootstrap status, capability flag |
| `CompanionContext` | 1 row | Current UI context mode |
| `CompanionSkill` | 0 initially | Coach-specific skills created through conversation (defaults are global) |
| `CompanionSchedule` | 0 initially | Created when coach requests scheduled actions |

### Default Skills (Global)

These ship with the platform (coach=null, available to all):

| Skill | Slug | Trigger |
|-------|------|---------|
| Session Prep | `session-prep` | `session_approaching` |
| Post-Session Debrief | `post-session-debrief` | `session_complete` |
| Hypothesis Check | `hypothesis-check` | `session_approaching`, `session_complete` |
| Client Re-engagement | `client-reengagement` | `client_inactive` |

Coaches customize these through conversation → creates a coach-scoped fork with `parent_skill` FK.

## Prompt Assembly Order

Every conversation turn, the prompt assembler builds the system prompt:

```
1. Personality layers (soul → identity → coach_knowledge → instructions → environment)
2. Capability state (what the companion can/can't do)
3. Active skills (matching the current trigger event)
4. Memory context (relevant coach profile, client knowledge, session intelligence)
5. Interface instructions (panel_updates, notifications, etc.)
6. Conversation history
```

**Token budgets:**

- Personality: ≤8,500 tokens total (per-layer caps)
- Skills: ≤8,000 tokens total (≤2,000 per skill)
- Memory: dynamic, based on relevance

## Data Model Relationships

```
CoachingProfile (existing)
  ├── CompanionPersonality (6 layers)
  ├── CompanionState (1 row)
  ├── CompanionContext (1 row)
  ├── CompanionSkill (0-N coach-specific)
  ├── CompanionSchedule (0-N)
  │     └── ScheduleExecution (audit log)
  ├── CompanionMemory (0-N)
  │     ├── CoachProfile entries
  │     └── ClientKnowledge entries (scoped by client)
  └── SessionIntelligence (1 per session, via CoachingSession FK)
```

All companion models use `coach` FK to `CoachingProfile`. Data is fully isolated per coach.

## Migration Strategy

Three phases, each independently rollbackable:

### Phase 1: Foundation (Weeks 1–4)

- Companion app scaffold + models + migrations
- Prompt assembler, chat API, SSE wiring
- Bootstrap flow, personality self-modification
- Menu item in portal UI

### Phase 2: Intelligence (Weeks 5–8)

- Session Intelligence model (replaces regenerated analysis)
- Session Analyzer modification (writes to both old + new models)
- Client Knowledge, scheduling engine
- Proactive behavior (prep, follow-up)

### Phase 3: Client-Facing (Weeks 9–12)

- Client portal companion integration
- Client-facing conversation API
- Automated client nudges
- Coach approval flow for client messages

**Rollback:** Each phase adds tables/routes removable without affecting existing functionality. During Phase 2, the session analyzer writes to both old and new models in parallel.

## Infrastructure Requirements

No new infrastructure. The companion uses everything already in the stack:

- **PostgreSQL** — new tables in existing DB
- **Redis** — existing cache + RQ queue
- **APScheduler** — new 5-minute polling job added to existing `clock.py`
- **django-eventstream** — new SSE channels for companion
- **RQ workers** — companion jobs on existing worker processes
- **Heroku** — same dynos, no new services

## OpenClaw Integration (Exploratory)

Marco is exploring using OpenClaw as the agent infrastructure — one OpenClaw agent per coach:

- Each coach gets their own OpenClaw workspace (SOUL.md = personality, MEMORY.md = coach knowledge)
- Django communicates via OpenClaw's OpenResponses HTTP API (`POST /v1/responses`)
- SSE streaming from OpenClaw to Django to browser
- Strong agent isolation (separate workspaces, sessions, auth profiles)

**Status:** Feasibility confirmed. Main gap is dynamic provisioning — OpenClaw requires config changes + gateway restart to add new agents.
