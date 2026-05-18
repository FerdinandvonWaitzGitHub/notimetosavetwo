Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Advisor (security + performance) — a Studio panel that reads `_ntts.*` health/audit tables and Postgres catalog to surface actionable issues. **Security checks**: tables without RLS; policies allowing `anon` unintentionally; un-MFA'd admin Operators; un-rotated API keys past threshold; permissive Realtime channel policies. **Performance checks**: slow Realtime policies (`realtime_uncached` marker per F-RT-7); top-10 slow SQL via `pg_stat_statements`; missing indexes flagged by `pg_stat_user_tables` heuristics; bloated tables (>20% dead tuples). Per-finding: severity, explanation, suggested action, "Dismiss" with reason.

## Acceptance criteria

- [ ] Table created without RLS surfaces in Advisor as ERROR
- [ ] Realtime policy uncacheable surfaces as WARN with `realtime_uncached` marker
- [ ] Top-10 slowest SQL stmts pulled from `pg_stat_statements` and shown with avg/max latency
- [ ] Dismissed finding records reason in `_ntts.advisor_dismissals`
- [ ] Findings refresh on demand + on a 5-min pg_cron schedule
- [ ] Role gates: viewer can read; only admin can dismiss

## Blocked by

- [25-realtime-postgres-changes.md](./25-realtime-postgres-changes.md)
- [07a-studio-shell-login-mfa-nav.md](./07a-studio-shell-login-mfa-nav.md)
