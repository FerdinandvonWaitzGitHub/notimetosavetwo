Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

`ntts upgrade [--to vX.Y]` orchestration per F-UPG-1..8 + ADR-0011. **Two migration streams**: app migrations in `migrations/` (Operator-owned, tracked in `_ntts.schema_migrations`) and NTTS-internal migrations in `_ntts/migrations/` embedded in the image (tracked in `_ntts.internal_migrations`). `_ntts.version (current_version, applied_at, applied_by)` marker; each release declares `MIN_KNOWN..MAX_KNOWN` compatibility range; boot refuses out-of-range. Upgrade flow: pre-check → auto-snapshot → image pull → upgrade-prep → upgrade-apply (in TX) → verify → HMAC auto-rotation (slice 4 hook). Failure before COMMIT → automatic compose-revert. Failure post-COMMIT → `ntts upgrade --rollback` restores from snapshot. `ntts upgrade --check` parses bundled `RELEASE_NOTES.md` + requires `--confirm-breaking` for breaking upgrades. Skip-version paths CI-tested. During upgrade, external traffic returns 503 with `Retry-After`; typical duration 1–2 min, acknowledged appliance downtime.

## Acceptance criteria

- [ ] `ntts upgrade --check --to v1.1` parses RELEASE_NOTES.md and lists breaking changes
- [ ] Breaking upgrade requires `--confirm-breaking`
- [ ] Full upgrade orchestrates all 6 phases + HMAC auto-rotation
- [ ] Failure pre-COMMIT → compose-revert; failure post-COMMIT → snapshot-restore via `ntts upgrade --rollback`
- [ ] External traffic returns 503 + `Retry-After` for duration of upgrade
- [ ] Boot refuses out-of-`MIN_KNOWN..MAX_KNOWN`-range with clear error
- [ ] Skip-version path (v1.0 → v1.3) validated by CI
- [ ] Audit-chain reset marker written for schema-mutating upgrades (slice 10 + F-STU-10)

## Blocked by

- [59-ci-cd-multi-arch-cosign.md](./59-ci-cd-multi-arch-cosign.md)
