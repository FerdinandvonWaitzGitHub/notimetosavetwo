Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire TextLocal (India-only) as a catalog-driven SMS provider (extension of slice 33a). Catalog manifest entry with `api_key`, `sender`, callback. `gotrue_env_mapping` for GoTrue's TextLocal provider. Surfaced with "India-only" badge in Studio so EU Operators don't pick it inadvertently.

## Acceptance criteria

- [ ] `ntts auth sms set textlocal` lands credentials encrypted
- [ ] `ntts auth sms test textlocal` sends a real OTP to an Indian number; result in audit log
- [ ] Studio form renders "India-only" badge prominently
- [ ] Reveal credentials requires admin + MFA re-auth

## Blocked by

- [33a-sms-catalog-twilio.md](./33a-sms-catalog-twilio.md)
