# Logs retention: two-class policy, partition-drop for function_logs, mirror-drain for audit, DB-resident alerting

NTTS distinguishes two classes of growing tables. `function_logs` is high-volume operational telemetry — it gets a hybrid time-and-size retention with daily range partitions and automated partition drops. The audit tables (`studio_audit_read`, `studio_audit_write`, `function_audit`) are low-volume tamper-evidence records — they get an optional mirror-drain to an external sink (`NTTS_AUDIT_DRAIN`) but never auto-delete locally. Local audit truncation is a separate, MFA-gated emergency operation that explicitly records a hash-chain reset. Disk pressure and drain lag are surfaced through a DB-resident alert engine that both Studio and CLI read from, plus a Prometheus-scrape endpoint for Operators who run external monitoring.

## Considered Options

- **Unified retention policy.** Rejected: applying time-based deletes to audit tables breaks the hash-chain tamper-evidence property (ADR-0009) silently.
- **Audit drain = mirror + auto-truncate.** Rejected: couples transport and destruction. A misconfigured drain sink would silently lose data when auto-truncate fires.
- **Bulk DELETE for function_logs.** Rejected: stunde-long locks, index bloat, vacuum overhead. Partition drop is O(1) DDL.
- **Passive alerting (external Prometheus only).** Rejected: contradicts the appliance promise. Operators must be able to detect disk pressure from inside the bundled stack.
- **Studio-resident alerting.** Rejected as primary: logic in studio-backend makes CLI and non-Studio consumers blind. DB-resident with a SQL table as source-of-truth lets everyone read the same state.

## Mechanism

### function_logs
- **Storage.** Declarative range-partitioned by `started_at` on day boundaries.
- **Retention triggers.** Hybrid:
  - **Time:** drop partitions older than `function_logs_retention_days` (default 30).
  - **Size cap:** drop oldest partitions until `pg_total_relation_size('_ntts.function_logs')` ≤ `function_logs_size_cap_gb` (default 10), even if within the time window.
- **Mechanism.** pg_cron `ntts_log_pruner()` runs hourly and reacts to a `system_alerts` CRITICAL row by running immediately (aggressive-prune mode).
- **Operator surface.** Env vars `NTTS_FUNCTION_LOGS_RETENTION_DAYS`, `NTTS_FUNCTION_LOGS_SIZE_CAP_GB` for bootstrap; live override via Studio Admin UI and `ntts logs retention set --days N --cap MGB`. Both land in `_ntts.config`.
- **Visibility.** Studio "Logs" panel shows `effective_oldest_log_at`; every size-cap-triggered prune writes a `_ntts.function_audit` row.

### Audit tables
- **Storage.** Range-partitioned by `created_at` on month boundaries (low volume).
- **Default retention.** Unbounded local.
- **Drain (optional).** `NTTS_AUDIT_DRAIN=s3://... | syslog://...` enables continuous mirror. Each successful sink-write writes a receipt to `_ntts.audit_drain_receipts(table_name, row_id, drained_at, sink_target)`.
- **Truncate.** Separate, MFA-gated, partition-aware command: `ntts audit truncate --before YYYY-MM-DD`. Refuses unless all rows older than the cutoff have receipts in `audit_drain_receipts`. Writes a `_ntts.audit_chain_resets(table_name, last_id_before, last_hash_before, reset_at, reset_by, drain_target)` row so `verify_audit_chain` knows the chain runs in segments.

### Alert engine
- **Table.** `_ntts.system_alerts (id uuid PK, severity enum('warn','error','critical'), kind text, payload jsonb, raised_at timestamptz, cleared_at timestamptz null)`.
- **Producer.** pg_cron `_ntts.check_health()` every 5 min computes disk-used pct, `function_logs` size vs cap, audit drain lag, PITR backup lag, partition count. Inserts new alerts, clears resolved ones, fires `NOTIFY ntts_health` on state change.
- **Default thresholds** (overridable via `_ntts.config`):
  - `disk_used_pct`: WARN 70, ERROR 85, CRITICAL 95.
  - `function_logs_size_pct_of_cap`: WARN 80.
  - `audit_drain_lag_minutes`: WARN 60, ERROR 1440 (only if drain configured).
  - `pitr_backup_lag_minutes`: WARN 30.
- **Severity actions.**
  - WARN: Studio header badge yellow, non-blocking.
  - ERROR: badge red, email to admin Operators (if SMTP), webhook (if configured).
  - CRITICAL: ERROR actions plus automatic mitigation — `function_logs` aggressive-prune; deploys and migrations not blocked, but recorded as "applied during CRITICAL alert" in audit.
- **Consumers.** Studio header polls `_ntts.system_alerts`. CLI `ntts health` reads same table. Prometheus endpoint `/metrics` on studio-backend exposes table sizes, drain lag, partition counts, current alert counts by severity.

## Consequences

- `_ntts.function_logs`, `_ntts.studio_audit_read`, `_ntts.studio_audit_write`, `_ntts.function_audit` become range-partitioned tables. The migration that introduces partitioning is part of `_ntts` internal migrations and is non-trivial — must be CI-tested on representative data volumes before any release.
- Audit truncation is a deliberate, audit-visible event. Operators inherit a workflow: ensure drain is healthy, run truncate with explicit cutoff, verify segment boundary in `audit_chain_resets`.
- Drain sinks must be Operator-managed (S3 lifecycle, syslog archive retention). NTTS guarantees the mirror, not the long-term storage policy of the sink.
- The Prometheus endpoint at `/metrics` is intentionally minimal — it is not a full APM surface, just enough for an Operator who already runs Prometheus to fold NTTS into existing dashboards.
- `_ntts.system_alerts` is itself non-audit data — it has no hash chain and is regularly pruned of cleared rows older than 30 days (built into `check_health()`).
- Disk pressure during CRITICAL never blocks read/write traffic; it only triggers aggressive-prune. The appliance prefers to keep serving and lose old telemetry over outage-as-protection.
