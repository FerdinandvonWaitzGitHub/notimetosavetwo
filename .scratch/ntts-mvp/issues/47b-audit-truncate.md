Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

`ntts audit truncate --before YYYY-MM-DD` is a separate, MFA-gated, partition-aware command (F-LOG-3). Refuses unless all rows older than cutoff have drain receipts (slice 47a). Writes `_ntts.audit_chain_resets (table_name, reset_at, reason, ntts_version_from, ntts_version_to)` row so `verify_audit_chain` reads the chain segment-wise (consistent with the schema-mutating-upgrade pattern in F-STU-10).

## Acceptance criteria

- [ ] `ntts audit truncate --before 2026-01-01` rejects if any pre-cutoff partitions lack drain receipts
- [ ] Successful truncate writes `_ntts.audit_chain_resets` row with `reason='operator_truncate'`
- [ ] `verify_audit_chain` reads chain segment-wise across the reset marker; reports OK for both segments
- [ ] Truncate requires admin + MFA re-auth
- [ ] Audit log of the truncate itself (paradox-safe: the truncate's audit row is in the post-cutoff partition)
- [ ] Partition-aware: drops whole partitions, not row-by-row

## Blocked by

- [47a-audit-drain-s3-syslog.md](./47a-audit-drain-s3-syslog.md)
