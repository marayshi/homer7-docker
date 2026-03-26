# Homer7 + Grafana + Prometheus + Loki — Production Stack

Production-ready SIP monitoring for a Kamailio SBC.

## Architecture

```
Internet
  │
  ├─ homer.example.com ──► Caddy (TLS) ──► Homer Web App ──► PostgreSQL
  ├─ grafana.example.com ─► Caddy (TLS) ──► Grafana ──────► Prometheus / Loki
  │
  └─ Kamailio (localhost) ──HEP/UDP──► heplify-server ──► PostgreSQL + Loki
```

## Prerequisites

- Docker Engine 24+ and Docker Compose v2
- DNS A records pointing to your VPS IP:
  - `homer.example.com`
  - `grafana.example.com`
- Ports 80 and 443 open on your firewall (for Caddy TLS)
- Kamailio configured to send HEP packets (see below)

## Quick Start

```bash
# 1. Copy and edit the environment file
cp .env.example .env
vim .env   # set HOMER_DOMAIN, POSTGRES_PASSWORD, GF_ADMIN_PASSWORD, SLACK_WEBHOOK_URL

# 2. Start the stack
docker compose up -d

# 3. Verify all services are healthy
docker compose ps
```

## Access

| Service       | URL                            | Credentials                  |
|---------------|--------------------------------|------------------------------|
| Homer         | https://homer.your-domain.com  | admin / sipcapture           |
| Grafana       | https://grafana.your-domain.com| admin / (from .env)          |
| Prometheus    | internal only (not exposed)    | —                            |
| Loki          | internal only (not exposed)    | —                            |
| Alertmanager  | internal only (not exposed)    | —                            |

## Environment Variables

See [.env.example](.env.example) for all configurable values:

| Variable              | Description                        | Default          |
|-----------------------|------------------------------------|------------------|
| `HOMER_DOMAIN`        | Base domain for Caddy TLS          | `example.com`    |
| `POSTGRES_USER`       | PostgreSQL username                | `homer`          |
| `POSTGRES_PASSWORD`   | PostgreSQL password                | **required**     |
| `GF_ADMIN_USER`       | Grafana admin username             | `admin`          |
| `GF_ADMIN_PASSWORD`   | Grafana admin password             | **required**     |
| `DB_DROP_DAYS`        | SIP data retention (days)          | `30`             |
| `PROMETHEUS_RETENTION`| Prometheus metrics retention       | `30d`            |
| `SLACK_WEBHOOK_URL`   | Slack incoming webhook for alerts  | **required**     |
| `SLACK_CHANNEL`       | Slack channel for alerts           | `#homer-alerts`  |

## Kamailio HEP Integration

Add to your `kamailio.cfg`:

```
loadmodule "siptrace.so"
modparam("siptrace", "duplicate_uri", "sip:127.0.0.1:9060")
modparam("siptrace", "hep_mode_on", 1)
modparam("siptrace", "hep_version", 3)
modparam("siptrace", "trace_on", 1)
modparam("siptrace", "trace_to_database", 0)
```

Then in your routing logic, call `sip_trace()` on the messages you want captured.

## Configuration Reload (no downtime)

```bash
# Prometheus
curl -s -XPOST http://localhost:9090/-/reload

# Alertmanager
curl -s -XPOST http://localhost:9093/-/reload
```

## Reset / Reinstall

To reset Homer and re-provision the database:
```bash
echo "" > ./bootstrap
docker compose restart homer-webapp
```

## Grafana Dashboards

Pre-provisioned dashboards:
- **SIP Overview** — ASR, NER, CPS, error rates
- **SIP KPIs** — SER/ASR, NER, SCR per RFC 6076
- **SIP Methods & Responses** — INVITE/REGISTER rates
- **SIP Calls & Registers** — concurrent sessions, CPS/RPS
- **SIP Error Rates** — 50x, 403, 404, 482 breakdowns
- **QOS RTCP / XRTP / Horaclifix** — MOS, jitter, packet loss, RTT
- **Host Overview** — CPU, memory, disk, network

## Alerts

Configured in `prometheus/alert.rules`, sent to Slack via Alertmanager:
- Service down (any monitored target)
- High CPU / memory / disk usage
- No HEP packets received (Kamailio not sending)
- SIP 50x server errors
