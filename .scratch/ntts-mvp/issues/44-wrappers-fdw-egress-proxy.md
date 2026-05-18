Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wrappers + FDW per ADR-0016 (F-FDW-1..7). Wrappers (Stripe, S3, BigQuery, Firebase, etc.) and `postgres_fdw` configured via `/etc/ntts/wrappers.catalog.json` (versioned per NTTS release). Studio renders setup forms dynamically; CLI `ntts wrappers add/test/disable/delete/export/apply`. Configuration in `_ntts.wrappers_config`. Credentials in Vault (slice 45). DDL synthesised by studio-backend. **New container `ntts-egress-proxy`** in compose stack. Postgres routes outbound HTTPS through proxy (env `https_proxy=…`). Allowlist in `_ntts.egress_allowlist`; wrapper enable auto-extends allowlist from catalog's `requires_egress_to`. Every connection logged to `_ntts.egress_log` (monthly-partitioned). Per-wrapper cost guardrails: `statement_timeout_ms` (30000), `max_result_rows` (10000), `monthly_query_count_cap`. Enforced via `_ntts.wrapper_safe_select_<table>()` SECURITY DEFINER wrappers; direct foreign-table queries bypass by design. Test: `ntts wrappers test <id>` runs catalog-declared smoke query; soft-warn if `last_tested_at` >30 days.

## Acceptance criteria

- [ ] `ntts-egress-proxy` container in compose; Postgres `https_proxy=` configured
- [ ] Direct outbound HTTPS from Postgres blocked unless host in `_ntts.egress_allowlist`
- [ ] `ntts wrappers add stripe ...` adds wrapper + auto-extends allowlist + DDL synthesised
- [ ] `ntts wrappers test stripe` runs catalog smoke query; result in `_ntts.wrappers_test_log`
- [ ] `wrapper_safe_select_<table>()` enforces statement_timeout + max_result_rows + monthly cap
- [ ] Every egress connection logs to `_ntts.egress_log`
- [ ] `ntts wrappers export > wrappers.toml` (credentials externalised) + `apply -f` round-trip
- [ ] Role gates: edit/test/disable/delete = admin + MFA; reveal Vault credential = admin + MFA re-auth (audit row)

## Blocked by

- [03-postgres-roles-three-tier.md](./03-postgres-roles-three-tier.md)
