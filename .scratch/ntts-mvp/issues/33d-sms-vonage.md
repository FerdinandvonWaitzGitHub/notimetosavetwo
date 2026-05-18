Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Vonage (NO/UK, EU-recommended) as a catalog-driven SMS provider (extension of slice 33a). Catalog manifest entry with `api_key`, `api_secret`, `from`, callback. `gotrue_env_mapping` for GoTrue's Vonage provider. EU-recommended status: surfaced as a green badge in Studio.

## Acceptance criteria

- [ ] `ntts auth sms set vonage` lands credentials encrypted
- [ ] `ntts auth sms test vonage` sends a real OTP; result in audit log
- [ ] Studio form renders fields dynamically; "EU-recommended" badge present
- [ ] Reveal credentials requires admin + MFA re-auth

## Blocked by

- [33a-sms-catalog-twilio.md](./33a-sms-catalog-twilio.md)
