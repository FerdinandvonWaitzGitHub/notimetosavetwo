Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Prometheus metrics endpoint on studio-backend (`/metrics`) exposing table sizes, drain lag, partition counts, alert counts by severity, `ntts_container_healthy`, `ntts_alert_open` (F-LOG-6). OTel exporter for the Edge Runtime (F-FN-17) configurable via env `OTEL_EXPORTER_OTLP_ENDPOINT`. JSON logs throughout with request-IDs.

## Acceptance criteria

- [ ] `GET /metrics` returns Prometheus exposition format
- [ ] Standard NTTS metrics present: `ntts_container_healthy`, `ntts_alert_open{severity}`, `ntts_function_logs_size_bytes`, `ntts_audit_drain_lag_seconds`, `ntts_partition_count{table}`
- [ ] Edge Runtime OTel exporter sends spans to configured OTLP endpoint
- [ ] All NTTS components emit JSON logs with `request_id` field
- [ ] Operator can scrape `/metrics` from external Prometheus

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
- [07a-studio-shell-login-mfa-nav.md](./07a-studio-shell-login-mfa-nav.md)
