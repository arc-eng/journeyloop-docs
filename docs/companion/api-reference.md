# Companion API

API reference for the JourneyLoop companion backend. All endpoints live under `/api/v1/companion/` and require authentication via the existing Django session/token auth.

---

## Chat

### POST /api/v1/companion/chat/

Send a message to the companion. Returns 202 (processing) — the actual response arrives via SSE.

**Request:**
```json
{
  "message": "How did my session with Marcus go?",
  "interaction_response": null
}
```

**Fields:**

- `message` (string) — Coach's text input
- `interaction_response` (object, optional) — Response to a scaffolded micro-interaction (multiple choice, yes/no, quick capture)

**Response:**
```json
{
  "status": "processing",
  "job_id": "rq-job-uuid"
}
```

**Flow:** Message is enqueued via django-rq → prompt assembler builds context (personality + skills + memory) → LLM generates `CompanionResponse` → response router publishes to SSE channels.

---

## SSE Event Stream

### GET /api/v1/companion/events/

Server-Sent Events stream for real-time panel updates. Uses django-eventstream. Each coach gets their own channel (`companion-{coach_id}`).

**Event types:**

| Event | Target | Description |
|-------|--------|-------------|
| `chat-message` | Chat panel | Companion's conversational reply |
| `main-update` | Main panel | Rich content push (prep cards, summaries, timelines) |
| `secondary-update` | Secondary panel | Supporting context (stats, schedule, related info) |
| `notification` | Overlay | Time-sensitive alert (8s auto-dismiss) |

**Each event payload:**
```json
{
  "html": "<rendered HTML fragment>"
}
```

Panels are HTMX-wired to swap content on the matching SSE event type.

---

## Panels

### GET /api/v1/companion/panels/{panel}/

Fetch current panel content (for initial page load or manual refresh).

**Parameters:**

- `panel` — `main` or `secondary`

**Response:**
```json
{
  "html": "<rendered HTML fragment>"
}
```

---

## Personality

### GET /api/v1/companion/personality/

Returns the companion's personality layer status (not raw content — the companion narrates its own state).

**Response:**
```json
{
  "bootstrap_status": "complete",
  "capabilities_unlocked": true,
  "layers": {
    "soul": { "is_set": true, "version": 3, "token_estimate": 450 },
    "identity": { "is_set": true, "version": 2, "token_estimate": 180 },
    "coach_knowledge": { "is_set": true, "version": 8, "token_estimate": 1200 },
    "instructions": { "is_set": true, "version": 1, "token_estimate": 600 },
    "environment": { "is_set": false, "version": 0, "token_estimate": 0 },
    "client_persona": { "is_set": false, "version": 0, "token_estimate": 0 }
  },
  "total_tokens": 2430,
  "token_budget": 8500
}
```

**Note:** Coaches never see raw layer content. This endpoint is for internal monitoring and test agents. The companion narrates its own personality state conversationally.

---

## Skills

### GET /api/v1/companion/skills/

List all active skills for the current coach.

**Query parameters:**

- `trigger` (optional) — Filter by trigger event (e.g., `session_approaching`, `session_complete`, `on_demand`)

**Response:**
```json
{
  "skills": [
    {
      "slug": "session-prep",
      "name": "Session Prep",
      "skill_type": "default",
      "trigger_events": ["session_approaching"],
      "token_estimate": 380,
      "version": 1,
      "is_customized": false
    },
    {
      "slug": "post-session-debrief",
      "name": "Post-Session Debrief",
      "skill_type": "custom",
      "trigger_events": ["session_complete"],
      "token_estimate": 420,
      "version": 3,
      "is_customized": true,
      "parent_slug": "post-session-debrief"
    }
  ],
  "total_tokens": 800,
  "max_context_tokens": 8000
}
```

### GET /api/v1/companion/skills/active/

Returns skills that would load for a specific trigger event.

**Query parameters:**

- `trigger` (required) — Trigger event name
- `coach_id` (optional) — For test agents

**Response:** Same shape as `/skills/` but filtered to matching triggers with coach overrides applied.

---

## Schedules

### GET /api/v1/companion/schedules/

List active schedules for the current coach.

**Response:**
```json
{
  "schedules": [
    {
      "id": 1,
      "name": "Sunday evening reflection",
      "schedule_type": "recurring",
      "cron_expression": "0 19 * * 0",
      "delivery_channel": "message",
      "is_paused": false,
      "last_triggered_at": "2026-02-16T19:00:00Z",
      "next_trigger_at": "2026-02-23T19:00:00Z"
    }
  ]
}
```

### GET /api/v1/companion/schedules/{id}/executions/

Audit log for a specific schedule.

**Response:**
```json
{
  "executions": [
    {
      "triggered_at": "2026-02-16T19:00:00Z",
      "status": "success",
      "output_summary": "Weekly reflection based on 3 sessions..."
    }
  ]
}
```

---

## Memory

### GET /api/v1/companion/memory/

Returns memory stats and recent entries (for monitoring and test agents).

**Query parameters:**

- `layer` (optional) — `coach_profile`, `client_knowledge`, `session_intelligence`
- `client_id` (optional) — Filter client knowledge by client

**Response:**
```json
{
  "stats": {
    "coach_profile_entries": 45,
    "client_knowledge_entries": 120,
    "session_intelligence_entries": 89
  },
  "recent": [
    {
      "layer": "session_intelligence",
      "session_id": 24,
      "summary": "Marcus brought up compensation directly...",
      "created_at": "2026-02-19T14:30:00Z"
    }
  ]
}
```

---

## Context

### GET /api/v1/companion/context/

Returns the companion's current UI context state.

**Response:**
```json
{
  "mode": "pre_session",
  "active_client": {
    "id": 5,
    "name": "Marcus Chen"
  },
  "updated_at": "2026-02-19T13:40:00Z"
}
```

### POST /api/v1/companion/context/

Force a context transition (for testing). Production transitions are event-driven.

**Request:**
```json
{
  "mode": "post_session",
  "client_id": 5
}
```

---

## Response Schema

All companion LLM responses use the `CompanionResponse` structured output schema:

```python
class CompanionResponse(BaseModel):
    chat_message: str                          # Conversational reply (always present)
    panel_updates: list[PanelDirective] = []   # Content pushes to Main/Secondary
    notification: Optional[str] = None         # Time-sensitive overlay text
    context_hint: Optional[str] = None         # Context mode transition hint
    personality_updates: list[PersonalityUpdate] = []  # Self-modification
    skill_updates: list[SkillUpdate] = []      # Skill CRUD
    schedule_updates: list[ScheduleUpdate] = [] # Schedule CRUD
```

The response router processes each field: chat message → SSE, panel updates → rendered HTML → SSE, personality/skill/schedule updates → DB writes.

---

## Authentication

All endpoints require Django session auth or token auth (existing portal authentication). No separate API keys — the companion API inherits the portal's auth system.

## Error Handling

Standard Django REST Framework error responses:

- `401` — Not authenticated
- `403` — Not authorized (e.g., accessing another coach's companion)
- `404` — Resource not found
- `422` — Validation error (e.g., skill exceeds token budget)
- `500` — Internal server error
