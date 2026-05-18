Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Fresh `docker compose up` brings up Postgres + `ntts-edge` Gateway + studio-backend skeleton + minimal Caddy on the `ntts-internal` network. Container healthchecks per F-HC-1..6 wire up; depends_on uses `service_healthy`. `_ntts` schema initialised from `apps/studio-backend/src/internal-migrations/0001_init.sql` (canonical DDL per F-UPG-1). studio-backend mounts `/var/run/docker.sock` read-only and writes `_ntts.container_status`. `_ntts.aggregate_health()` returns GREEN when all containers healthy + pg_cron alive + no open alerts.

## Acceptance criteria

- [ ] `docker compose up` reaches GREEN aggregate-health in <60s on a 4-vCPU/8-GB VPS
- [ ] Gateway is the only container with public ports (80/443); Postgres `5432` only exposed if `NTTS_EXPOSE_POSTGRES=true`
- [ ] `ntts status` CLI command returns GREEN
- [ ] Gateway `/healthz` returns 200 + JSON; `/healthz?strict=true` returns 503 for YELLOW/RED
- [ ] Prometheus `/metrics` on studio-backend exposes `ntts_container_healthy` + `ntts_alert_open`
- [ ] 3-consecutive-unhealthy → auto-restart + ERROR alert; 3 restarts/10min → CRITICAL, no further auto-restart
- [ ] Postgres unhealthy raises ERROR immediately; restart requires admin + MFA confirmation

## Blocked by

None — can start immediately.
