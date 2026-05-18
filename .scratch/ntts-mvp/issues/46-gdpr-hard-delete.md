Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

GDPR hard-delete endpoint `/admin/gdpr/delete-user` (F-GDPR-1..4). Accepts App User id. Wipes `auth.users` row; foreign-key cascade through application tables driven by Operator's declared map `_ntts.gdpr_cascade (table_name, key_column, action enum('delete','redact'))`. Function logs containing the user's id are tombstoned (row kept, payload redacted). Audit logs (`studio_audit_*`) are **not** wiped — they record Operator actions. PITR backups older than retention (default 30 days) are not purged; Operator runs the redaction script on backup restore. CLI `ntts users delete <id>` is the same endpoint.

## Acceptance criteria

- [ ] `POST /admin/gdpr/delete-user` with App User id wipes `auth.users` row (§6 #24)
- [ ] Declared cascade tables redacted/deleted per `_ntts.gdpr_cascade` rules
- [ ] `_ntts.function_logs` rows referencing the user are tombstoned (payload redacted, row kept)
- [ ] Audit rows preserved (referential integrity to `_ntts.studio_users` intact)
- [ ] Redaction script for backup restore exists + documented in runbook
- [ ] CLI `ntts users delete <id>` routes through the same endpoint with audit
- [ ] Endpoint requires admin + MFA re-auth

## Blocked by

- [09-rest-rls.md](./09-rest-rls.md)
- [10-audit-chain-integrity.md](./10-audit-chain-integrity.md)
