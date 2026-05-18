Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire MessageBird (NL, EU-recommended) as a catalog-driven SMS provider (extension of slice 33a). Catalog manifest entry with `access_key`, `originator`, callback. `gotrue_env_mapping` for GoTrue's MessageBird provider. EU-recommended status: surfaced as a green badge in Studio.

## Acceptance criteria

- [ ] `ntts auth sms set messagebird` lands credentials encrypted
- [ ] `ntts auth sms test messagebird` sends a real OTP; result in audit log
- [ ] Studio form renders fields dynamically; "EU-recommended" badge present
- [ ] Reveal credentials requires admin + MFA re-auth

## Blocked by

- [33a-sms-catalog-twilio.md](./33a-sms-catalog-twilio.md)
