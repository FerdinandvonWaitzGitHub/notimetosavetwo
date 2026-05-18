Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Phone-OTP sign-in (F-AUTH-2): 6-digit code via SMS, 5-min expiry, max 3 verify attempts. supabase-js `auth.signInWithOtp({ phone })` + `auth.verifyOtp({ phone, token, type: 'sms' })`. Routed through the configured SMS provider (slice 33a + fan-out).

## Acceptance criteria

- [ ] `auth.signInWithOtp({ phone: '+49…' })` sends SMS via configured provider
- [ ] `auth.verifyOtp(...)` returns JWT on correct code
- [ ] 3 wrong attempts lock out re-verify
- [ ] Code expires in 5 min
- [ ] Phone-only user `GET /auth/v1/user` returns `email: null, phone: '+49…'` (§6 #3)

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
- [33a-sms-catalog-twilio.md](./33a-sms-catalog-twilio.md)
