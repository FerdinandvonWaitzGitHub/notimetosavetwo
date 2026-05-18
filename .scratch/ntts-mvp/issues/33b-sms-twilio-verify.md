Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Twilio Verify as a catalog-driven SMS provider (extension of slice 33a). Catalog manifest entry with `account_sid`, `auth_token`, `verify_service_sid`, callback. `gotrue_env_mapping` for GoTrue's Twilio Verify provider. EU-recommended status: US-hint (same as Twilio).

## Acceptance criteria

- [ ] `ntts auth sms set twilio_verify` lands credentials encrypted
- [ ] `ntts auth sms test twilio_verify` sends a real OTP via Twilio Verify; result in audit log
- [ ] Studio form renders fields dynamically from catalog manifest
- [ ] Reveal credentials requires admin + MFA re-auth

## Blocked by

- [33a-sms-catalog-twilio.md](./33a-sms-catalog-twilio.md)
