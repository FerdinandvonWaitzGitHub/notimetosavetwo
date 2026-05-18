Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Forward-only migrations from `migrations/NNNN_slug.sql` (Operator-owned, source-of-truth in app repo). Each runs in a transaction by default; `-- ntts:no-transaction` pragma opts out. `_ntts.schema_migrations (version, name, hash, applied_at, applied_by)` tracks applied state. File-hash drift detection at next run. After successful COMMIT, fire `NOTIFY pgrst, 'reload schema'`; control-plane polls PostgREST `/openapi` every 50ms up to 5s and `ntts db migrate` returns only after verification (ADR-0021). Non-migration DDL (raw psql, Function-code) accepts a ≤2s stale-cache window; SDK retries PGRST200 with backoff (100/300/900ms; max 3). Events logged to `_ntts.schema_cache_events`; WARN alert if >5% of trailing-24h events exceed `latency_ms=3000`. `ntts db status` compares structural hash of current schema vs latest applied migration.

## Acceptance criteria

- [ ] `ntts db migrate` applies `migrations/NNNN_slug.sql` files in order; transaction per file unless pragma opts out
- [ ] `_ntts.schema_migrations` carries `(version, name, hash, applied_at, applied_by)`
- [ ] File-hash drift detected and warned at next `ntts db migrate` run
- [ ] After COMMIT, control-plane polls PostgREST `/openapi` and returns only after schema-cache reload verified
- [ ] `_ntts.schema_cache_events` populated; WARN alert when >5% of last-24h events exceed 3000ms
- [ ] No down-migrations supported; `ntts db make <slug>` scaffolds new forward migration
- [ ] Role gates: `admin` + `developer` can apply; `viewer` can read; UI Apply buttons disabled for viewer
- [ ] `ntts db status` reports manual schema drift via structural hash compare

## Blocked by

- [02-setup-token-first-admin.md](./02-setup-token-first-admin.md)
