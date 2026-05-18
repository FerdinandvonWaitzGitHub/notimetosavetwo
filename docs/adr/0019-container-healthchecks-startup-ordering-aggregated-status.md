# Container healthchecks, startup ordering, aggregated GREEN/YELLOW/RED status

Every container in the NTTS compose stack carries a Docker `healthcheck:` block hitting a service-specific liveness endpoint. Startup ordering uses `depends_on: condition: service_healthy` so a stack comes up in the right sequence, deterministically. An NTTS-internal watcher in studio-backend reads Docker's health state every 30 seconds and reacts: three consecutive unhealthy checks trigger one auto-restart plus an ERROR alert; three restarts in ten minutes escalate to CRITICAL with no further auto-restart, requiring Operator action. Postgres is exempted from auto-restart and instead surfaces a confirm prompt. An aggregated status (GREEN/YELLOW/RED) combines container health, open alerts, pg_cron liveness, TLS expiry, and PITR lag into a single signal — exposed through `ntts status`, the Studio Health page, and the Gateway `/healthz` endpoint for external load-balancers.

## Considered Options

- **No healthchecks; rely on Docker restart on crash.** Rejected: process-up ≠ service-functional. PostgREST and GoTrue can run with the listener open but the upstream Postgres reachable only intermittently — Docker sees them up; clients see 500s.
- **Compose-only health, no NTTS-internal watcher.** Rejected: Compose marks unhealthy but does not act. The appliance promise includes self-healing under bounded failure modes.
- **Aggressive auto-restart on every unhealthy.** Rejected: restart-loops on Postgres are dangerous (corruption risk during WAL replay loops); restart-loops on stateful services in general can mask root causes.
- **`/healthz` returns 200 only when fully GREEN.** Rejected as default: YELLOW means degraded but still serving. Operators can opt into strict mode via `?strict=true` for external probes that should fail closed.

## Mechanism

### Per-container healthchecks
- All containers carry `healthcheck:` with `interval: 10s`, `timeout: 3s`, `retries: 3`, `start_period: 30s`.
- Test commands (one per container):

| Container | Test |
|---|---|
| `ntts-edge` (Caddy) | `wget -q --spider http://localhost:2019/config/` |
| `postgres` | `pg_isready -U postgres -h /var/run/postgresql` |
| `supavisor` | `nc -z localhost 4000 && wget -q --spider http://localhost:4000/api/health` |
| `postgrest` (REST + pg_graphql) | `wget -q --spider http://localhost:3000/` |
| `gotrue` | `wget -q --spider http://localhost:9999/health` |
| `supabase-storage` | `wget -q --spider http://localhost:5000/status` |
| `supabase-realtime` | `wget -q --spider http://localhost:4000/api/health` |
| `supabase-edge-runtime` | `wget -q --spider http://localhost:9000/_internal/health` |
| `studio-backend` | `wget -q --spider http://localhost:8000/healthz` |
| `ntts-typegen` | `wget -q --spider http://localhost:8001/healthz` |
| `ntts-egress-proxy` | `wget -q --spider http://localhost:3129/health` (admin port, separate from `:3128` CONNECT port) |

### Startup ordering
- `postgres` boots first.
- `supavisor`, `ntts-egress-proxy` wait on `postgres` (egress-proxy could start parallel but waiting simplifies the dep graph).
- `postgrest`, `gotrue`, `supabase-storage`, `supabase-realtime`, `supabase-edge-runtime`, `ntts-typegen` wait on `supavisor` (Postgres reachability).
- `studio-backend` waits on all of the above.
- `ntts-edge` waits on `studio-backend` (the last public-facing entry).
- All `depends_on` use `condition: service_healthy`.

### Unhealthy handling
- `restart: unless-stopped` on every container.
- NTTS-internal watcher inside studio-backend polls `docker ps --format json --filter health=unhealthy` every 30s.
- **3 consecutive unhealthy checks** on a container → studio-backend issues `docker restart <container>` and writes an ERROR alert to `_ntts.system_alerts` with `kind='container_unhealthy'`, `payload={container, last_logs_tail}`.
- **3 restarts within 10 minutes** → CRITICAL alert, no further auto-restart, Operator action required. UI surfaces with "Acknowledge and Retry" or "Investigate" controls.
- **Postgres is exempt from auto-restart.** Unhealthy Postgres surfaces an ERROR alert immediately and requires Operator confirmation in Studio (admin + MFA) before restart is allowed — protects against WAL-replay loops or other data-layer cascades.

### Aggregated status
- **GREEN.** All containers `healthy`; `_ntts.system_alerts` has no open ERROR or CRITICAL; pg_cron health job ran in the last 10 minutes; TLS expiry > 7 days for all domains; PITR lag < 30 min (when WAL-G is active).
- **YELLOW.** ≥1 open WARN alert, all other GREEN criteria met. Still serving.
- **RED.** ≥1 open ERROR or CRITICAL alert, OR ≥1 container unhealthy, OR pg_cron stale > 10 min.
- Computed by a SQL function `_ntts.aggregate_health()` joining container status (read via a small docker-socket-reading shim in studio-backend that writes `_ntts.container_status (name, state, health, last_started_at, restart_count_10m, updated_at)` every 30s), `system_alerts`, `pg_cron.job_run_details` last row, `tls_cert_status`, and PITR lag.

### Surfaces
- **`ntts status`** CLI: prints aggregate + per-container table + open alerts.
- **Studio "Health" page:** all of the above + Restart Container + Force Healthcheck Now actions (MFA-gated for Postgres).
- **Gateway `/healthz`:**
  - Default: returns `200` with `{status: 'GREEN' | 'YELLOW' | 'RED', containers, alerts, last_evaluated_at}`. Body always JSON; status code reflects RED-as-503 only.
  - Strict mode `/healthz?strict=true`: `200` only for GREEN, `503` for YELLOW or RED. For external probes that should fail closed on degraded.
- **Prometheus `/metrics` (F-LOG-6):** exposes `ntts_container_healthy{name=…}` gauge, `ntts_alert_open{severity=…}` gauge.

## Consequences

- A docker-socket-reading shim runs inside studio-backend; this requires mounting `/var/run/docker.sock` read-only into the studio-backend container. Documented as a privilege requirement; alternative is a tiny "docker-spy" sidecar — same security shape.
- The dependency graph means a cold `ntts up` takes longer than a parallel start. The trade is determinism and Operator clarity over startup speed. Cold-start time is documented as part of NFRs.
- Postgres-no-auto-restart shifts the responsibility for data-layer recovery onto a human in the loop. This is the intended trade — the appliance prefers a paged Operator over a corrupted database.
- The aggregated status function (`_ntts.aggregate_health()`) is the single canonical answer to "is NTTS up?" — every UI / CLI / external probe reads it. Future signals (e.g., AI-provider health, S3-drain health) plug into the same function.
- Strict-mode `/healthz` lets Operators behind a load-balancer treat YELLOW as down and drain traffic; default-mode keeps serving so end-users aren't kicked off by a non-critical WARN alert.
- The `_ntts.container_status` table is non-audit operational data; partitioned daily and pruned aggressively (24h retention) since it's high-frequency-write.
