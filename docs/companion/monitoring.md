---
title: Monitoring Stack
description: Prometheus + Grafana observability for the companion operator — metrics, dashboards, and access.
---

# :material-chart-line: Monitoring Stack

Prometheus + Grafana monitoring for the companion operator on GCP. Covers message throughput, token cost, session health, and VM/container resource usage.

---

## Services

| Service | Purpose |
|---------|---------|
| `otel-collector` | Receives OTLP/HTTP metrics from OpenClaw; exposes Prometheus scrape endpoint |
| `prometheus` | Scrapes otel-collector, cAdvisor, node-exporter; 30-day retention |
| `grafana` | Dashboard UI — SSH tunnel access only, no auth required |
| `cadvisor` | Docker container CPU/memory metrics |
| `node-exporter` | GCP VM CPU/memory/disk metrics |

All services run via Docker Compose in `~/journeyloop/operator/monitoring/` on the GCP VM.

---

## Dashboard

The **Companion Operator** Grafana dashboard is auto-provisioned. It covers:

- **Message flow** — processed rate by outcome, p50/p95 processing duration, webhook errors
- **Token usage & cost** — tokens/min by type, USD/hr by model
- **Sessions & queue** — state transitions, stuck session count, queue depth
- **Docker containers** — CPU and memory per container
- **GCP VM** — CPU %, memory %, disk usage %

---

## Accessing Grafana

Grafana is bound to `127.0.0.1:3000` — not exposed to the internet.

```bash title="SSH tunnel"
ssh -L 3000:localhost:3000 mlamina@34.63.156.77
```

Then open [http://localhost:3000](http://localhost:3000) — no login required.

---

## Operations

```bash title="Start the stack"
cd ~/journeyloop/operator/monitoring
docker compose up -d
docker compose ps
```

```bash title="Check logs"
docker compose logs -f otel-collector
docker compose logs -f prometheus
```

```bash title="Stop / wipe"
docker compose down          # stop services
docker compose down -v       # stop + wipe all data
```

---

## OpenClaw Configuration

OpenClaw must push OTLP metrics to `http://localhost:4318`. Add to `~/.openclaw/openclaw.json` on the GCP VM:

```json
{
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://localhost:4318",
      "protocol": "http/protobuf",
      "serviceName": "openclaw-companion-operator",
      "traces": false,
      "metrics": true,
      "logs": false,
      "flushIntervalMs": 30000
    }
  },
  "plugins": {
    "allow": ["diagnostics-otel"],
    "entries": {
      "diagnostics-otel": { "enabled": true }
    }
  }
}
```

Then reload the gateway:

```bash
sudo systemctl reload openclaw-gateway
# or: docker compose kill -s USR1 openclaw-gateway
```
