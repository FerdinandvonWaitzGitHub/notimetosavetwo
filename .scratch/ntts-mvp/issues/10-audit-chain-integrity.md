Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Audit tables (`_ntts.studio_audit_read`, `studio_audit_write`, `function_audit`) are append-only per ADR-0009. `GRANT INSERT, SELECT` only on audit tables to all application roles. `BEFORE UPDATE OR DELETE` trigger raises exception. Hash-chain (`prev_hash` + `payload_hash`) per row; INSERTs serialised via `pg_advisory_lock`. `_ntts.verify_audit_chain(table_name)` SQL function walks each table segment-wise across `_ntts.audit_chain_resets` markers. CLI `ntts audit verify` reports chain health. Studio "Audit Health" panel surfaces chain status + last-verified timestamps. Audit tables range-partitioned monthly (F-LOG-2); never auto-deleted.

## Acceptance criteria

- [ ] Audit tables grant INSERT + SELECT only; UPDATE/DELETE attempts raise exception
- [ ] Hash-chain on every row: `payload_hash = sha256(payload)`, `prev_hash = previous row's payload_hash`
- [ ] `pg_advisory_lock` serialises INSERTs per audit table
- [ ] `_ntts.verify_audit_chain(table_name)` returns OK / error with offending row id
- [ ] `ntts audit verify` runs verify across all audit tables; non-zero exit on chain breakage
- [ ] Studio "Audit Health" panel surfaces verify status + last-verified timestamps
- [ ] Audit tables range-partitioned monthly; partition creation is automatic ahead of month-end

## Blocked by

- [07b-studio-audit-write-read-every-page.md](./07b-studio-audit-write-read-every-page.md)
