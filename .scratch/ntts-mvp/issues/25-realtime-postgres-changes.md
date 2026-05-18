Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

supabase-realtime (Phoenix, Elixir fork) wrapped behind Gateway. Postgres-changes (logical replication) broadcasts require explicit per-table opt-in via `_ntts.realtime_enabled_tables (schema, table, enabled_at, enabled_by)`; pre-toggle UI check verifies RLS active + at least one policy. Per row-change × per subscriber, Realtime sets the subscriber's JWT context and runs a PK `SELECT` against the changed row; zero rows → skip broadcast, non-zero → broadcast returned columns. **Three-Layer Authorization per ADR-0026**: Layer-1 topic-filter bucketing + Layer-2 Subscriber-Authorization-Snapshot-Cache (evicted on `NOTIFY ntts_jwt_changed` and `NOTIFY ntts_policy_changed`) + Layer-3 bulk-RLS-eval fallback. F-RT-6 HMAC trust via `hmac_gateway_realtime` (Realtime trusts `X-NTTS-Auth` from `ntts-edge`, its own JWT-verify disabled). Service-JWT subscribers bypass RLS as normal Postgres semantics dictate. Denies sampled into `_ntts.realtime_policy_denies` (rate-limited, retention via F-LOG-1 pattern).

## Acceptance criteria

- [ ] `.channel().on('postgres_changes', …).subscribe()` works unchanged (§6 #13)
- [ ] 1.000 Subscribers on a table with RLS `user_id = auth.uid()`, 50 INSERTs/s by users X1..X50: each subscriber sees only own rows; Postgres load <500 RLS-queries/s in `pg_stat_statements` (§6 #26)
- [ ] HMAC `hmac_gateway_realtime` validates; Realtime's own JWT-verify disabled
- [ ] Cache-hit-rate ≥80% under the §7 5.000-subscriber capacity-target with simple RLS
- [ ] Service-JWT subscribers bypass RLS
- [ ] `_ntts.realtime_policy_denies` populated, rate-limited
- [ ] Pre-toggle check warns if RLS off / no policies on target table

## Blocked by

- [04-internal-hmac-bootstrap.md](./04-internal-hmac-bootstrap.md)
- [09-rest-rls.md](./09-rest-rls.md)
