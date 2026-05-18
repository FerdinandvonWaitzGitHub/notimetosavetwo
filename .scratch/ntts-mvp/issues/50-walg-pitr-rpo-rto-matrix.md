Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

WAL-G PITR (wrap) with S3 destination + Operator-configured retention (default 30 days). `ntts backup pitr-status` reports lag, last-archived LSN, oldest restore point. `ntts backup restore --to <timestamp>` restores to PITR. RPO/RTO matrix (§7): PITR active → RPO 1 min, RTO 30 min. Pre-flight check at PITR-enable; alert on backup-lag. Backup-lag WARN 30min per F-LOG-4.

## Acceptance criteria

- [ ] `ntts backup pitr-status` reports last-archived LSN + lag
- [ ] PITR restore to 30 min ago succeeds (§6 #23)
- [ ] PITR restore loses ≤60s of data in standardised load-test (§7 RPO 1min)
- [ ] RTO ≤30 min for typical-size restore measured in CI
- [ ] Pre-flight check at PITR-enable validates S3 destination + creds; refuses if misconfigured
- [ ] Backup-lag >30min triggers WARN alert

## Blocked by

- [01-compose-greens-up.md](./01-compose-greens-up.md)
