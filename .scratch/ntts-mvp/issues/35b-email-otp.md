Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Email-OTP sign-in: 6-digit code via email, 5 min expiry, max 3 verify attempts. supabase-js `auth.signInWithOtp({ email, options: { shouldCreateUser: true } })` then `auth.verifyOtp({ email, token, type: 'email' })`.

## Acceptance criteria

- [ ] `auth.signInWithOtp({ email })` sends a 6-digit code
- [ ] `auth.verifyOtp(...)` with correct code returns JWT
- [ ] Wrong code 3× locks out re-verify until a new code is requested
- [ ] Code expires in 5 min
- [ ] Per-locale email template

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
