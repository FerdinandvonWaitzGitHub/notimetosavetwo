# Upgrades via `ntts upgrade`, two migration streams, version-range compatibility

NTTS upgrades are an orchestrated operation, not a naive `docker compose pull`. The CLI command `ntts upgrade [--to vX.Y]` snapshots the DB, pulls images, applies an internal `_ntts` migration in a transaction, restarts services, and verifies health. App migrations and NTTS-internal migrations are separate streams. Each NTTS version declares a compatibility window over `_ntts` schema versions so an Operator can skip several releases in one jump, provided the skip path is CI-tested.

## Considered Options

- **Naive `compose pull` (Option A).** Rejected: silently breaks on schema changes, no rollback path, no pre-flight check.
- **Blue-green stacks (Option C).** Rejected for the appliance: requires two full stacks, contradicts "single-node appliance".
- **Compatibility window only (Option D).** Rejected alone: doesn't address the workflow ergonomics or rollback.
- **Orchestrated CLI + compatibility window + two migration streams (chosen).** Operator runs one command; the tool handles snapshot, prep, apply, verify, with explicit rollback if any step fails before COMMIT.

## Mechanism

- **Two migration streams:**
  - **App migrations** (`migrations/NNNN_*.sql`, source-of-truth in the App repo) — tracked in `_ntts.schema_migrations`.
  - **NTTS-internal migrations** (`_ntts/migrations/NNNN_*.sql`, embedded in the NTTS image) — tracked in `_ntts.internal_migrations`.
- **Version marker** `_ntts.version (current_version, applied_at, applied_by)`. Each NTTS release declares a *supported range* (`MIN_KNOWN..MAX_KNOWN`). Boot refuses to start if marker is out-of-range, pointing the Operator to `ntts upgrade`.
- **Upgrade command flow:**
  1. `ntts upgrade --check` — diff release notes, list breaking changes, dry-run migration. Standalone-runnable.
  2. Auto-snapshot (`ntts backup snapshot`) before any change.
  3. `docker compose pull`.
  4. New image runs `_internal upgrade-prep` (validates marker is in-range, generates internal migration SQL).
  5. New image runs `_internal upgrade-apply` (applies internal migration in a TX, starts services).
  6. `ntts upgrade --verify` (health, smoke, audit-chain validation, function boot-hydration check).
- **Snapshot-ordering invariant:** Step 2's snapshot MUST capture the database in a state from which the OLD image can successfully boot. This means **all version-state mutations (including the `_ntts.version` marker update, all `_ntts.internal_migrations` writes, all platform_secrets re-encryptions, all schema changes affecting audit chain) MUST happen exclusively inside the step-5 transaction.** No write to `_ntts.*` is allowed in steps 1–4. Concretely: `upgrade-prep` is *read-only* against `_ntts.*` — it may write to ephemeral files but never to the database. The rollback contract depends on this: after `ntts upgrade --rollback`, the OLD image must boot against the restored snapshot without out-of-range marker errors.
- **Failure handling:** before COMMIT in step 5 → automatic compose-revert. After COMMIT but verify-failure → `ntts upgrade --rollback` restores the snapshot from step 2.
- **Master-key-required migrations:** when an internal migration must re-encrypt `_ntts.platform_secrets` or `_ntts.function_secrets`, the migration manifest declares `requires_master_key: true`. Master-key materialisation during upgrade-apply follows [ADR-0028](./0028-master-key-availability-during-upgrade-apply.md).
- **Edge-Runtime bundle-format migration:** when the upgrade brings an Edge Runtime version with an incompatible eszip-format range, behaviour follows [ADR-0027](./0027-edge-runtime-bundle-format-migration-over-upgrade.md). Bundles outside the supported range mark the affected Functions `requires_redeploy` rather than crashing the upgrade.
- **Audit-chain reset on schema-mutating upgrades:** if an internal migration alters the column set or canonical hash input of any `_ntts.studio_audit_*` or `_ntts.function_audit` table, the migration writes an `_ntts.audit_chain_resets` row in the same transaction (`reason='ntts_upgrade'`, `ntts_version_from`, `ntts_version_to`). `_ntts.verify_audit_chain()` reads chains segment-wise across these markers per F-LOG-3.
- **Postgres-major** is a separate path (`ntts upgrade --postgres-major`) requiring explicit Operator confirmation; runs `pg_dump`/restore against the new Postgres major.

## Consequences

- `B0..B6` in §11 Roadmap are *engineering milestones to first public v1.0*, not Operator-facing versions. Operator-facing semver starts at v1.0.
- Skip-version paths must be CI-tested. The test matrix grows linearly with releases; small-team discipline assumed.
- Release notes are part of the upgrade UX, not optional documentation. `ntts upgrade --check` parses them.
- Live Function traffic during upgrade is 503'd, typically 1–2 minutes. Documented as accepted appliance downtime; clients-with-retry assumed.
- A Blue/Green upgrade story exists only as a future Cluster Mode feature, not in the appliance promise.
