Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Custom SMTP via env-vars (F-AUTH-5). Email templates per locale stored in `_ntts.auth_templates` (DE + EN default, Operator can add). **Engine for Auth-mails: Go `text/template`** (GoTrue native — Auth mails flow through GoTrue). Studio editor shows Go-template syntax with variable docs. **NTTS-internal mails** (Operator-Invite, Health-Alerts, Backup-Reports) use **Eta** in studio-backend (separate table `_ntts.ntts_email_templates`). MJML/React-Email deferred post-v1.

## Acceptance criteria

- [ ] SMTP env-vars configured → password reset email sent
- [ ] `_ntts.auth_templates` carries per-locale Go-template strings for each auth email type
- [ ] Studio template editor highlights Go-template syntax + shows variables doc panel
- [ ] DE locale used when Operator/App-User locale is `de`; EN fallback otherwise
- [ ] Operator-Invite mail rendered from `_ntts.ntts_email_templates` via Eta in studio-backend
- [ ] Health-alert + backup-report mails use the same Eta path
- [ ] Save reload propagates within 2s via NOTIFY for both engines

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
