Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Capacity smoke-test in CI (small) + full scale-test as a separate workflow (manual + nightly). §7 capacity targets: Concurrent App-User-Sessions 10.000 / Deployed Functions 500 / Realtime concurrent Subscribers 5.000 / Tables in `public.*` 1.000 / Storage FS-Objects 1.000.000 / API RPS sustained 100, burst (<60s) 500 / Function-Invocations 50 RPS / active pg_cron 100. API P95 <100ms at <100 RPS on 4 vCPU / 8 GB VPS. Acceptance §6 #26: 1.000 Realtime subscribers + 50 INSERTs/s with simple RLS keeps Postgres <500 RLS-queries/s in `pg_stat_statements`.

## Acceptance criteria

- [ ] CI smoke-test runs against sub-targets (e.g. 50 subscribers, 5 RPS) on every main-merge
- [ ] Full scale-test workflow (`.github/workflows/scale-test.yml`) runs nightly + on-demand against full targets
- [ ] §6 #26 verified: 1.000 subscribers + 50 INSERTs/s → <500 RLS-queries/s
- [ ] API P95 <100ms at <100 RPS on 4 vCPU / 8 GB benchmark VM
- [ ] Burst 500 RPS for 60s sustained without 5xx
- [ ] Results published to a fixed location in repo for trend tracking

## Blocked by

- [25-realtime-postgres-changes.md](./25-realtime-postgres-changes.md)
- [55-js-sdk-client.md](./55-js-sdk-client.md)
