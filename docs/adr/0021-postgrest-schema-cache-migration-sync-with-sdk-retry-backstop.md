# PostgREST schema-cache: migration-side verify-and-wait, SDK retry backstop, documented non-migration window

The 200ms–2s window between a DDL commit and PostgREST's schema cache catching up is closed differently for the canonical migration path versus everything else. For migrations, the control-plane fires `NOTIFY pgrst`, then polls PostgREST's `/openapi` endpoint up to 5 seconds to verify the expected schema change is reflected — `ntts db migrate` returns only after verification (or timeout-with-warning). For raw DDL outside the migration path (Function-code DDL, Operator psql via the pg_event_trigger backstop), no synchronization is possible because there's no orchestrating caller; the stale-cache window is documented and the SDK retries PGRST stale-cache errors with exponential backoff as a defence-in-depth backstop. Every reload event is recorded in `_ntts.schema_cache_events` for Operator visibility, with a WARN alert if latency exceeds threshold across rolling samples.

## Considered Options

- **Accept the window, document only.** Rejected as the only line: `ntts db migrate && curl /rest/v1/new_table` failing in CI is a real friction point that the control-plane is well-positioned to eliminate for the canonical path.
- **Gateway blocks REST/GraphQL during reload.** Rejected: introduces a tail-latency spike for unrelated requests; couples the data-plane to a control-plane signal; PostgREST has no native "reload-complete" callback so the gate-open trigger is its own polling problem anyway.
- **Pre-warm PostgREST schema cache before DDL commit.** Rejected: requires forking or patching PostgREST; not viable for an upstream-wrap product.
- **SDK retry only.** Rejected as the only line: SDK retries shift the cost onto every caller and are silent — the migration tool can guarantee freshness explicitly and should.

## Mechanism

### Migration-path verify-and-wait
1. Migration transaction COMMITs.
2. Control-plane fires `NOTIFY pgrst, 'reload schema'` in the post-migration chain (same chain as F-MIG-5 / ADR-0012's typegen NOTIFY).
3. Control-plane inserts `_ntts.schema_cache_events (id, schema_epoch, notified_at, verified_at, latency_ms, source enum('migration','ddl_backstop'))` with `source='migration'`, `verified_at=null`.
4. Control-plane polls PostgREST `/openapi` (or `/`) every 50ms. Polling compares the response against an expected-diff derived from the migration's AST — at minimum, presence of new table/view names and absence of dropped ones; column-level checks for ALTER TABLE statements.
5. On match, control-plane sets `verified_at = now()` and `latency_ms = verified_at - notified_at`. `ntts db migrate` returns success to the caller.
6. On 5-second timeout without match, control-plane sets `latency_ms=5000`, leaves `verified_at` null, raises a WARN alert (F-LOG-4, kind=`schema_cache_reload_timeout`), and proceeds — does not block migration completion forever on a PostgREST quirk.

### Non-migration path (ddl_backstop source)
- `pg_event_trigger` (ADR-0012) fires `NOTIFY pgrst` for raw DDL outside the migration path.
- No verification poll (no orchestrating caller to block on it). Inserts `_ntts.schema_cache_events` row with `source='ddl_backstop'`, `verified_at=null`.
- Documented as "REST/GraphQL may serve stale schema for ≤2s after non-migration DDL"; this is the acknowledged trade for Operator power-user flexibility.

### SDK retry backstop
- NTTS client SDK (TypeScript + mobile SDKs per ADR-0005-adjacent surfaces) detects PGRST200-class errors ("Could not find … in the schema cache") by error code, not message text.
- Retries with backoff: 100ms, 300ms, 900ms; max 3 attempts; only for the specific PGRST stale-cache codes. Other 4xx/5xx surfaces immediately.
- Documented in SDK docs as a stale-cache safety net, not a general retry policy.

### Operator visibility
- Studio "Schema cache events" panel (under the Database section): last N events as a table with `latency_ms`, `source`, `verified` status, latency-distribution chart.
- Alert engine (F-LOG-4) raises WARN `kind=schema_cache_slow` when more than 5% of events in the trailing 24h have `latency_ms > 3000`. Suggests "review schema size / PostgREST instance sizing".

## Consequences

- A new low-volume operational table `_ntts.schema_cache_events`; partitioned monthly, retention via F-LOG-1 pattern.
- Migration tooling becomes slightly slower under happy path (typical 200ms verify-poll added to migration completion), which is overwhelmingly preferred over the alternative ("migrate returned green but my next REST call 404'd").
- The verify-poll uses PostgREST's `/openapi` endpoint — if a future PostgREST version makes this endpoint expensive, polling cost grows. Mitigation: keep polling rate at 50ms intervals (modest), cap polling to 5s total.
- Non-migration DDL window stays open as a documented caveat. Operators who absolutely need synchronous reload for raw DDL can use `SELECT pg_notify('pgrst', 'reload schema');` followed by their own client-side wait — same primitive, no NTTS support.
- SDK retry backstop is silent by default; verbose-logging mode surfaces a counter so callers can see when they're hitting stale-cache retries (useful during migration-storm CI runs).
- Latency-distribution alerting requires aggregation queries against `schema_cache_events`; cheap given low volume, but the alert engine implementation needs to support it.
