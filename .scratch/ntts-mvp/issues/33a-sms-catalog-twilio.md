Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

SMS provider catalog per F-AUTH-11 (analogue of F-AUTH-10). **Catalog**: `/etc/ntts/sms-providers.catalog.json` lists GoTrue's native providers; Studio renders setup form dynamically from the schema. **Storage**: `_ntts.auth_sms_provider (provider_id, config_json, secrets_vault_ref, enabled_at)`; credentials pgcrypto-encrypted (studio_secrets class). **Reload**: Studio-Backend rendert `/var/run/ntts/gotrue.env` neu, `NOTIFY ntts_sms_change` → GoTrue reload (same pipeline as OAuth reload). **Role gates**: edit/test/disable = admin + MFA re-auth; view without secrets = admin + developer. **EU-Sovereignty-Empfehlung**: MessageBird (NL), Vonage (NO/UK) marked "EU-recommended"; Twilio remains available with US-hint; TextLocal marked "India-only". CLI: `ntts auth sms set <provider>`, `ntts auth sms test <provider>`, `ntts auth sms disable`. **Twilio** wired as first provider (the tracer). Remaining providers fan out across slices 33b..33e.

## Acceptance criteria

- [ ] Catalog manifest loaded at boot; Studio renders form dynamically
- [ ] `ntts auth sms set twilio` accepts credentials, lands them encrypted in `auth_sms_provider`
- [ ] `ntts auth sms test twilio` sends a real OTP to Operator's phone; result in audit log
- [ ] Studio shows "EU-recommended" badge for MessageBird + Vonage; "US" hint for Twilio; "India-only" for TextLocal (badge plumbing — providers themselves arrive in fan-out)
- [ ] Reveal credentials requires admin + MFA re-auth
- [ ] NOTIFY-driven GoTrue reload happens in <2s

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
