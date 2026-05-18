Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Storage backup integration per ADR-0022 and F-STOR-10. **FS-backend**: `ntts backup snapshot` produces `pg_dump.tar.gz` + `caddy.tar.gz` + `storage.tar.gz` with bounded drift (rows created during the tar window may be metadata-only); restore via `ntts backup restore`. `--check-storage` re-hashes all objects vs metadata SHA-256 (slice 27); mismatches → exit 2 + list in `_ntts.storage_integrity_failures`. **S3-backend**: snapshot skips storage tar; Studio Backup page surfaces "S3 bucket backup is your responsibility (configured: …)". **Override**: `NTTS_BACKUP_SKIP_STORAGE=true` for Operators using external backup tooling. (Incremental snapshots are v1.1+ roadmap.)

## Acceptance criteria

- [ ] `ntts backup snapshot` FS-mode produces 3 tarballs in expected directory
- [ ] `ntts backup restore --check-storage` re-hashes objects and reports mismatches via exit 2 + `storage_integrity_failures` rows
- [ ] S3-backend snapshot skips storage tar; Studio Backup page renders the warning
- [ ] `NTTS_BACKUP_SKIP_STORAGE=true` skips storage tar with explicit log line
- [ ] Snapshot completes within 30 min for 1M objects (§7 capacity)
- [ ] Restore is bit-exact relative to non-drift rows

## Blocked by

- [27-storage-core.md](./27-storage-core.md)
