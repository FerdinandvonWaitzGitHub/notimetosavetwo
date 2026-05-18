Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Backup encryption default-on per §10-Q2-Resolution. Separate `NTTS_BACKUP_KEY` (not `NTTS_MASTER_KEY`). WAL-G via libsodium. `pg_dump` piped through [age](https://age-encryption.org). Both sub-keys derived from `NTTS_BACKUP_KEY` via HKDF. Opt-out via `NTTS_BACKUP_ENCRYPTION=false` for Operators with already-encrypted backup storage. Boot-check: if encryption on + key missing → WARN alert + red banner in Studio Backup page. Key-escrow recommendation documented in Operator docs.

## Acceptance criteria

- [ ] `ntts backup snapshot` produces age-encrypted `pg_dump.tar.gz.age`
- [ ] WAL-G PITR archives encrypted via libsodium
- [ ] Decrypt + restore succeeds with correct `NTTS_BACKUP_KEY`; fails clearly with wrong key
- [ ] HKDF-derived sub-keys deterministic given the same root key
- [ ] `NTTS_BACKUP_ENCRYPTION=false` skips encryption with explicit log line
- [ ] Boot-check fires WARN + red banner if encryption on + key missing
- [ ] Operator docs include key-escrow recommendation

## Blocked by

- [49-pg-dump-snapshot.md](./49-pg-dump-snapshot.md)
- [50-walg-pitr-rpo-rto-matrix.md](./50-walg-pitr-rpo-rto-matrix.md)
