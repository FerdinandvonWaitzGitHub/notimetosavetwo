Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

`ntts backup snapshot` produces a coherent snapshot of the Deployment: `pg_dump.tar.gz` (logical dump) + `caddy.tar.gz` (TLS state from slice 48a) + `storage.tar.gz` (FS storage from slice 29). `ntts backup restore` reverses it. Coordination ensures `pg_dump` finishes before `storage.tar` starts so the metadata-rows-without-objects drift window is minimised (bounded drift acknowledged in F-STOR-10).

## Acceptance criteria

- [ ] `ntts backup snapshot` produces three tarballs with hash-checksums
- [ ] `ntts backup restore` to a fresh Postgres → backend wakes up with all Functions, auth configs, buckets, Operators exactly as before (§6 #12)
- [ ] `pg_dump` + storage tar coherence verified (no orphan storage objects after restore)
- [ ] `ntts backup snapshot --check-storage` after restore re-hashes all storage objects + reports drift
- [ ] Snapshot completes within 30 min for a 50GB DB + 1M storage objects (§7 capacity-target)

## Blocked by

- [27-storage-core.md](./27-storage-core.md)
- [48a-tls-le-http01-backup.md](./48a-tls-le-http01-backup.md)
