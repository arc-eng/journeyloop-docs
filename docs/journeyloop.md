# JourneyLoop — Platform Reference for Companion Agents

## What is JourneyLoop?

JourneyLoop is an AI-powered coaching companion platform for independent coaches. Its tagline: *"You hold space for everyone. But who holds space for you?"*

The core premise: coaching is an isolated profession. JourneyLoop fixes that by acting as a companion to the coach — capturing every session, surfacing insights, reflecting growth, and preparing coaches for what's next. Coaches don't manually take notes or review transcripts. JourneyLoop handles that automatically.

**The product does three things:**
1. **Records and transcribes** coaching sessions automatically (via Recall.ai bot joining Zoom/Meet/Teams)
2. **Analyzes transcripts** using the SHIFT framework — giving coaches structured feedback on their coaching quality
3. **Tracks client progress** across goals, milestones, and behavior patterns over time

Coaches get AI-generated session notes, ICF-aligned coaching feedback, and cumulative client progress tracking — without any manual input.

---

## Core Data Model

### Coach (`CoachingProfile` → `auth.User`)
The primary user. Has a name, business name, coaching niche, and description. Owns all clients and sessions.

### Client (`CoachingClient`)
Belongs to one coach. Key fields:
- `name`, `email` — identity
- `total_sessions` — sessions in the coaching package (e.g., 6 or 12)
- `actual_completed_sessions` — dynamic count (date ≤ today, type = regular)
- `sessions` → `CoachingSession` objects
- `progress_goals` → `CoachingGoal` objects
- `milestones` → `CoachingMilestone` objects
- `has_consented` — client must consent before AI can process their transcripts
- Optional client portal access (linked `auth.User`)

### Session (`CoachingSession`)
A single coaching meeting. Key fields:
- `client` — FK to `CoachingClient`
- `title`, `date`, `session_type` (`regular` or `intake` — intake doesn't count toward package total)
- `transcript` — full text (deferred by default for performance; use `all_fields` manager when needed)
- `transcript_json` — raw JSON from Recall.ai with timestamps
- `meeting_url`, `recall_bot_id`, `recall_bot_status` — Recall.ai recording integration
- `progress_extracted` — whether AI has processed this session
- `is_completed` — property: `date <= today`

### Utterance (`SessionUtterance`)
Individual speech segment from a transcript:
- `speaker_type` — `coach` or `client`
- `text_clean` — cleaned text
- `sequence_number` — 1-based ordering
- `timestamp_min` / `timestamp_sec` — position in session

---

## The SHIFT Framework

SHIFT is JourneyLoop's proprietary coaching quality model, grounded in ICF competencies and evidence-based coaching research. After each session, AI analyzes the transcript against 5 principles, identifying moments where the coach applied them well (**Excellence Signals**) or missed an opportunity (**Growth Opportunities**).

**SHIFT = S·H·I·F·T**

### S — Surface Emotion
*Notice, name, and validate client emotions when they arise.*

**Patterns (what good looks like):**
- **Label the Feeling** — Coach reflects client's emotional language (*"That overwhelm sounds really heavy for you."*)
- **Ask About Emotion** — Coach directly inquires about feelings (*"What feelings are coming up as you say that?"*)
- **Affective Mirroring** — Coach's tone matches client's emotional state

**Anti-patterns (missed opportunities):**
- **Task-Switch after Emotion** — Pivoting to tasks right after emotional disclosure
- **Emotion Avoidance** — Ignoring or sidestepping emotional cues
- **Affect-Free Paraphrase** — Paraphrasing content without reflecting the emotion

---

### H — Hand Back Ownership
*Return agency to the client. Let them define direction and next steps.*

**Patterns:**
- **Invite Decisions** — Coach asks client to define their own solutions
- **Agency Language** — Framing that emphasizes the client's choice (*"It's your choice which step you take first."*)
- **Curiosity over Advice** — Using inquiry instead of giving answers

**Anti-patterns:**
- **Preemptive Advice** — Offering solutions before the client has a chance to find their own
- **Goal Takeover** — Coach reshapes or restates the client's goals with their own agenda
- **Directed Solutions** — Dictating what the client should do

---

### I — Illuminate Contradictions
*Spot and reflect inconsistencies, limiting beliefs, or avoidance in the client's language.*

**Patterns:**
- **Name the Tension** — Coach reflects contradictions in client statements (*"You say you want to take risks, yet uncertainty feels intolerable — how do those fit together?"*)
- **Values-Behavior Check** — Highlighting misalignment between stated values and actions
- **Invite the Disowned** — Exploring topics the client is avoiding

**Anti-patterns:**
- **Contradiction Blindness** — Ignoring clear inconsistencies in the client's narrative
- **Comfort Detour** — Sidestepping difficult or uncomfortable material
- **Reassure Instead of Explore** — Soothing away contradictions rather than exploring them

---

### F — Find the Pattern
*Name meaningful themes or behavioral loops visible within (and across) the session.*

**Patterns:**
- **Theme Detection** — Identifying a recurring theme in the client's language
- **Name Core Driver** — Identifying a deeper belief or value driving behavior
- **Reframe to Meta** — Lifting the narrative to a higher pattern-based level

**Anti-patterns:**
- **Surface Parroting** — Repeating client words without deeper synthesis
- **Theme Avoidance** — Ignoring recurring signals
- **Reactive Only** — Responding moment-to-moment without connecting the dots

---

### T — Turn Insight into Action
*Help the client translate reflection into clear, specific, committed next steps.*

**Patterns:**
- **Elicit Concrete Action** — Asking the client to define their own next steps (*"What's one specific action you'll take this week?"*)
- **Make It Measurable** — Encouraging time-bound, specific commitments
- **Reflect Decisions Made** — Consolidating commitments voiced during the session

**Anti-patterns:**
- **No Next Step** — Session ends without any explicit action
- **Insight-Only Closure** — Celebrating insight but skipping action planning
- **Coach-Defined Action** — Prescribing steps instead of eliciting them

---

## AI-Generated Session Content

When a transcript is uploaded, a `SessionContentRequest` processes four things asynchronously:

1. **Session Analysis** (`CoachingSessionAnalysis`) — overall session summary, themes, key takeaways
2. **SHIFT Insights** — `GrowthOpportunity` + `ExcellenceSignal` objects (each linked to specific utterances, with feedback to the coach, why it matters, and a better example)
3. **Progress** — `SessionObservation` objects extracted from utterances, linked to goals and milestones with numeric delta impact
4. **Behavior Loops** (`ClientLoop`) — recurring trigger-response-consequence patterns identified across sessions

---

## Progress Tracking

### Goals (`CoachingGoal`)
Long-running objectives. Have a `slug` (unique per client, used for AI reference), `title`, `status` (`active`/`paused`/`complete`). Progress accumulates via `ObservationGoal` delta links (-0.10 to +0.10 per observation).

### Milestones (`CoachingMilestone`)
Date-driven events (e.g., "Sales presentation on March 15th"). Have `target_date`, `status` (`upcoming`/`completed`/`missed`).

### Observations (`SessionObservation`)
AI-extracted client moments. `stance` = `struggle`/`intent`/`plan`/`reflection`/`outcome`. Include valence (-1.0 to 1.0), intensity, and certainty scores. Linked to goals and milestones with delta impacts.

### Behavior Loops (`ClientLoop`)
Cross-session patterns: `trigger` → `response` → `consequence`. Status: `emerging`/`active`/`evolving`/`resolved`.

---

## Companion API (`/api/companion/v1/`)

The agent's data access layer. Bearer token auth (`CompanionAPIKeyAuthentication`), scoped to the coach who owns the key.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/clients/` | List clients (id, name, email) |
| `GET` | `/clients/<id>/` | Client detail with goals and milestones |
| `GET` | `/clients/search/?q=` | Search clients by name |
| `GET` | `/clients/<id>/sessions/` | All sessions for a client |
| `GET` | `/clients/<id>/goals/` | List goals |
| `POST` | `/clients/<id>/goals/` | Create a goal |
| `PATCH` | `/clients/<id>/goals/<goal_id>/` | Update goal title or status |
| `DELETE` | `/clients/<id>/goals/<goal_id>/` | Delete a goal |
| `GET` | `/sessions/upcoming/` | Upcoming sessions (next 7 days) |
| `GET` | `/sessions/recent/?limit=N` | Recent past sessions (default 10) |
| `GET` | `/sessions/<id>/` | Session detail |
| `GET` | `/sessions/<id>/transcript/` | Raw transcript text |
| `PATCH` | `/profile/` | Update companion profile (e.g., set name) |

---

## Key Integrations

- **Recall.ai** — recording bot that joins video calls, captures live transcript
- **Google Calendar** — auto-creates sessions from calendar events
- **Stripe** — coach subscription billing (Starter free, paid tiers for more clients)
- **Client portal** — clients can log in, view session takeaways, track their own progress
