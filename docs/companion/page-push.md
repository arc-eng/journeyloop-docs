---
title: Companion Page Push
description: How the companion pushes UI pages to the coach and directs coach attention during sessions.
---

# :material-monitor-share: Companion Page Push

The companion can push pages from the JourneyLoop UI directly into the coach's view. This turns the companion from a chat interface into an active collaborator — surfacing the right information at the right moment, rather than waiting for the coach to navigate there.

---

## What Gets Pushed

The companion can push four page types:

| Page | What it shows |
|------|--------------|
| **Client profile** | Client background, goals, relationship context |
| **Session view** | A specific coaching session and its notes |
| **Calendar** | Scheduling and upcoming sessions |
| **Observations** | Patterns and progress observations for a client |

The companion decides what to push based on conversation context — for example, pushing the session view when a coach asks about a recent call, or the client profile when they're about to start a new engagement.

---

## Push Panel Design

When the companion pushes a page, it appears in an **overlay panel** alongside the chat — not by shrinking the chat window.

- The panel has a **draggable divider** between chat and content, so coaches control how much space each gets
- Closing the panel returns to full-width chat
- The **last pushed page persists across session reloads** — if a coach refreshes the browser mid-session, the panel reopens to where they left off

The overlay model was chosen over a layout shift (shrinking chat) because coaches are mid-conversation when pages are pushed. Collapsing the chat to make room would disrupt the flow. The draggable divider gives control without forcing a tradeoff.

---

## Spotlight: Directing Coach Attention

The companion can also initiate **highlights** (spotlight) — directing the coach's attention to a specific UI element within the pushed page. This lets the companion say, in effect, "look at this specific thing."

Spotlight is companion-initiated, not coach-triggered. The companion uses it when it's identified something specific worth examining — not as a general navigation tool.

---

## Part of the Companion's Identity

Page push is woven into the companion's `SOUL.md`, not treated as a bolt-on tool. The companion understands the push panel as part of how it communicates — it can show, not just tell.

This distinction matters: a companion that treats page push as a feature will use it awkwardly. One that treats it as a communication channel will use it the way a good co-pilot uses a shared screen.
