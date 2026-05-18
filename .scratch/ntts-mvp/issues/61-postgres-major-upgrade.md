Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Postgres-major upgrade path per F-UPG-6: separate `ntts upgrade --postgres-major` with explicit Operator confirmation and longer accepted downtime. Uses `pg_upgrade` against the Postgres data volume + Master-Key availability during upgrade-apply (ADR-0028 considerations). Detailed runbook published.

## Acceptance criteria

- [ ] `ntts upgrade --postgres-major --to 17` runs `pg_upgrade` against data volume
- [ ] Master-Key (`NTTS_MASTER_KEY`) remains available for the duration so audit/secret encryption stays operational
- [ ] Longer downtime accepted + announced to Operator pre-start
- [ ] Failure path: rollback via snapshot restore documented + tested
- [ ] Extension version compatibility checks pre-start
- [ ] Audit chain reset marker written

## Blocked by

- [60-ntts-upgrade-orchestration.md](./60-ntts-upgrade-orchestration.md)
