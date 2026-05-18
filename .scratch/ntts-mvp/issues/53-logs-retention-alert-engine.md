Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Two-class logs retention per ADR-0013. **`_ntts.function_logs` hybrid retention** (F-LOG-1): time-based `NTTS_FUNCTION_LOGS_RETENTION_DAYS` (30) AND size-cap `NTTS_FUNCTION_LOGS_SIZE_CAP_GB` (10); pg_cron `ntts_log_pruner()` hourly drops oldest day-partitions when either bound trips. **Audit tables never auto-deleted** (slice 10 already partitions). **Alert engine** (F-LOG-4): pg_cron `_ntts.check_health()` every 5 min computes disk-used %, function_logs size, drain lag, PITR lag, partition counts; inserts/clears `_ntts.system_alerts`; `NOTIFY ntts_health` on state change. Default thresholds: disk WARN 70% / ERROR 85% / CRITICAL 95%; drain-lag WARN 60min / ERROR 1440min; PITR-lag WARN 30min. **Severity actions** (F-LOG-5): WARN → Studio header yellow badge; ERROR → red badge + email + webhook; CRITICAL → ERROR actions + automatic aggressive prune. Traffic never blocked by disk pressure. **Live config** (F-LOG-7): `ntts logs retention set --days N --cap MGB`, `ntts health threshold set <kind> <value>`; persisted in `_ntts.config`.

## Acceptance criteria

- [ ] `function_logs` hybrid prune drops day-partitions when either time- or size-bound trips
- [ ] `check_health()` runs every 5min; populates `_ntts.system_alerts`
- [ ] Studio header yellow/red badges driven by alerts
- [ ] Email + webhook fire on ERROR (when SMTP / webhook URL configured)
- [ ] CRITICAL triggers aggressive prune; traffic still flows
- [ ] `ntts logs retention set` + `ntts health threshold set` persist to `_ntts.config` + NOTIFY-reload
- [ ] `/metrics` exposes alert counts by severity

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
- [52-prometheus-otel.md](./52-prometheus-otel.md)
