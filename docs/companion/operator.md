# Companion Operator

How the companion is architected, provisioned, and operated.

---

## Architecture

Each coach gets their own AI agent running inside OpenClaw — an open-source agent infrastructure platform. These agents run on a dedicated GCP VM (the "companion operator") rather than on Heroku with the Django app.

**Why separate infrastructure:**
- Agent memory, sandboxing, and tool execution need persistent processes — not stateless Heroku dynos
- Docker sandbox isolation per coach: each agent runs in its own container with zero access to other coaches' data
- OpenClaw handles conversation history, compaction, tool routing, and LLM calls natively

**The two-layer model:**
```
Django (Heroku)          ←→       OpenClaw (GCP VM)
  Coach UI                          Main operator agent
  Companion API                     Per-coach sandboxed agents
  CompanionProfile model            journeyloop CLI in sandbox
```

The Django app proxies chat to OpenClaw via `POST /v1/responses` with `x-openclaw-agent-id`. OpenClaw handles the rest.

---

## Provisioning Flow

When a coach is provisioned from the Django admin:

1. Django's `CompanionProvisioningService` sends an instruction to the main operator agent
2. The operator runs `provision-coach.sh` — creates the agent in OpenClaw config, writes workspace files (SOUL.md, TOOLS.md, AGENTS.md), generates an API key
3. The API key is registered with Django via `POST /api/companion/v1/internal/set-key/<agent_id>/` — Django stores only the SHA-256 hash, never the plaintext
4. The agent's `.credentials.json` (API key + Django URL) is written to the workspace

**Why operator-generated keys:** Django used to generate keys and store them. After a Copilot review flagged this, the architecture was inverted — the operator generates the key and registers the hash with Django. This way the plaintext never exists in Django's database.

---

## Key Design Decisions

**OpenClaw as infrastructure (not custom agent stack)**
We evaluated building a custom companion agent stack (personality model, memory system, scheduling engine). Decision: use OpenClaw instead. SOUL.md = personality, MEMORY.md = memory, cron = scheduling. Eliminated months of custom infrastructure work.

**One agent per coach, not a shared agent**
Each coach gets their own isolated OpenClaw agent. This means separate conversation history, separate memory, separate workspace. Coaches never share context.

**Sandbox isolation**
Each coach agent runs in a Docker container (`companion-sandbox:latest`) with:
- Bridge networking (can reach the internet but not the host network)
- `workspaceAccess: rw` for the agent's own workspace only
- Exec allowed for `journeyloop` CLI only; browser, gateway, web_search, image all denied

**Generated CLI over hand-written client**
The `journeyloop` CLI inside the sandbox is generated from the Django OpenAPI spec via `openapi-python-client`. When the API changes, rebuild the sandbox image. This keeps the CLI and API in sync without manual maintenance.

---

## Operations

**GCP VM:** `34.63.156.77` — SSH as `mlamina`

**Key services on the VM:**
- `openclaw-gateway.service` — OpenClaw gateway, runs natively (not Docker), port 18789
- `cloudflared.service` — Cloudflare tunnel (ephemeral URL, changes on restart)

**Companion operator repo:** `arc-eng/companion-operator` — provisioning scripts, Dockerfile, templates

**Django staging:** `https://staging.journeyloop.ai` — `OPENCLAW_GATEWAY_URL` must point to the current tunnel URL. Update whenever the tunnel restarts.

**To rebuild the sandbox image after an API change:**
```
ssh mlamina@34.63.156.77
cd ~/openclaw-operator
./scripts/build-sandbox.sh
```
Then restart the OpenClaw gateway so it picks up the new image.
