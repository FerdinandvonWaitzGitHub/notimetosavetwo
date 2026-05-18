Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Audit-write and audit-read hooks fire on every Operator action in Studio. Every write action lands `_ntts.studio_audit_write` with `(prev_hash, payload, payload_hash, diff)` (PII redacted in `diff`). Every PII-read by an Operator (opening `auth.users` list, reading a log row containing App-User data) lands `_ntts.studio_audit_read` with `(studio_user_id, action, target_kind, target_id, accessed_at)` — the fact of access, not the content. Gateway audit-hook (F-GW-7) routes studio-route audit-writes based on route metadata; Studio Frontend records UI-side PII reads via explicit telemetry calls. Audit-write fires on Operator-Invite + role-change + Pipeline override + Function deploy + migration apply + secret rotation.

## Acceptance criteria

- [ ] Opening `auth.users` list lands one row in `_ntts.studio_audit_read` (§6 #20)
- [ ] Every write Operator action lands one row in `_ntts.studio_audit_write` with diff
- [ ] `payload` of audit-read is the fact-of-access, never the read content
- [ ] PII redacted from `studio_audit_write.diff`
- [ ] Hash-chain `prev_hash` + `payload_hash` correctly serialised via `pg_advisory_lock`
- [ ] Audit-hook routes from F-GW-7 cover all `/studio/api/...` write routes
- [ ] BEFORE UPDATE OR DELETE trigger on audit tables raises exception (append-only enforced at DB level)

## Blocked by

- [07a-studio-shell-login-mfa-nav.md](./07a-studio-shell-login-mfa-nav.md)
