# Storage FS backup: snapshot-included with bounded drift, S3-backend operator-managed

`ntts backup snapshot` produces three tarballs in a single atomic-enough operation: `pg_dump`, `caddy.tar` (per ADR-0017), and `storage.tar` of `/data/storage`. The Storage-FS volume is captured between pg_dump start and end, so any `storage.objects` row created before the storage tar is guaranteed to have its bytes in the tarball. Rows created in the narrow window between the storage tar and pg_dump completion can be metadata-only — an accepted, documented drift that matches the same failure mode S3 backends exhibit under concurrent writes. Operators using the S3 backend opt out of the storage tar entirely; S3 bucket backup is their responsibility, the Studio Backup page surfaces this clearly. Per-bucket quotas and disk-used alerts close the operational loop on the appliance.

## Considered Options

- **Separate `ntts backup storage` command.** Rejected: contradicts the "one command snapshot, one command restore" appliance promise. Operator must remember two commands; one will inevitably be forgotten.
- **Force S3 backend in production.** Rejected: contradicts the appliance positioning ("works out of the box on a single node"). Operators who want zero infrastructure outside the box must have a FS path that backs up cleanly.
- **Continuous mirror to MinIO sidecar.** Rejected for v1.0: introduces a new container and a new failure mode for marginal benefit. Snapshots cover the use case.
- **Strict consistency (lock all uploads during snapshot).** Rejected: snapshot windows can run minutes on large stores; locking uploads for that long is a worse operational property than tolerating bounded drift.

## Mechanism

### Snapshot composition
- `ntts backup snapshot` produces `<timestamp>-pg_dump.tar.gz`, `<timestamp>-caddy.tar.gz`, `<timestamp>-storage.tar.gz`, plus a manifest `<timestamp>-manifest.json` declaring the three parts, their checksums, and the timestamps.
- Sequence:
  1. Begin `pg_dump` (records start_timestamp).
  2. `rsync` (or `tar`) `/data/storage` → `<timestamp>-storage.tar.gz` (records storage_tar_start, storage_tar_end).
  3. Finish `pg_dump` (records end_timestamp).
  4. `tar /data/caddy` → `<timestamp>-caddy.tar.gz`.
  5. Compute checksums, write manifest.
- Consistency guarantee: every `storage.objects` row with `created_at < storage_tar_start` has its bytes in the tar. Rows in `[storage_tar_start, pg_dump_end]` may be metadata-only.

### Restore
- `ntts backup restore <snapshot>` reverses the order: restore Storage FS, then load pg_dump, then restore Caddy.
- Optional `ntts backup verify --check-storage` post-restore: scans `storage.objects` for rows whose file is missing from `/data/storage`, prints the list. Operator decides: drop the orphan rows, or leave them (Storage returns 404 gracefully on missing bytes).

### S3-backend mode
- When `STORAGE_BACKEND=s3`: `/data/storage` is unused, `storage.tar` is small/empty.
- Studio "Backup" page surfaces the backend status:
  - FS: "Storage included in snapshot — `<size>` MB at last snapshot."
  - S3: "Storage backend: S3 (bucket `<name>`, region `<region>`). NTTS does not back up S3 buckets — configure lifecycle/replication on the bucket."
- `ntts backup snapshot` short-circuits storage tarring when backend is S3.

### Operator override
- `NTTS_BACKUP_SKIP_STORAGE=true` env skips the storage tar on FS backend (for Operators using external storage backup tooling). Warning printed each snapshot run.

### Disk health and per-bucket quotas
- `/data/storage` is included in the F-LOG-4 `disk_used_pct` calculation. WARN/ERROR/CRITICAL thresholds apply.
- Per-bucket quotas enforced at upload time via `storage.buckets.file_size_limit` and an added `object_count_limit` column. When breached, upload rejected with a 413 / quota-specific error code.
- Quota status visible in Studio Storage panel per bucket; CLI `ntts storage buckets quota` exposes the same.

## Consequences

- Snapshot duration grows linearly with `/data/storage` size. Documented in NFRs; large-store Operators are pointed at S3 backend or the future `--incremental` flag.
- The bounded-drift window means a post-restore application might briefly see `storage.objects` rows whose `GET` returns 404. The application's storage layer must tolerate 404 on download — already a normal Storage behaviour, not a new requirement.
- An S3-backend Operator who forgets to configure bucket lifecycle/replication gets no NTTS-side warning. The Studio Backup page surfaces this in plain language but doesn't probe S3 to verify lifecycle exists. This is the natural NTTS/Operator boundary: NTTS owns its own data plane; S3 buckets are external.
- `--incremental` snapshots are a v1.1+ roadmap item; v1.0 ships full snapshots only. For Operators with 100GB+ stores this means snapshot operations are infrequent (overnight, not hourly).
- Per-bucket quota enforcement adds a column to `storage.buckets`; a `_ntts` migration adds it. Existing buckets get `object_count_limit=null` (unbounded) by default — backwards-compatible.
- The Storage-FS-included snapshot pattern brings the three-tarball restore path to mainline; restore tooling must handle partial-restore (e.g., restore only pg_dump if storage tar is unavailable for any reason) gracefully.
