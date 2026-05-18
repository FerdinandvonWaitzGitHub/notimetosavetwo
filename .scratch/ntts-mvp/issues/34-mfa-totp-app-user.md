Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

GoTrue MFA TOTP for App Users (separate from Operator MFA in slice 02). Enroll/verify/disable flow exposed via supabase-js + `/auth/v1/factors` endpoints. Recovery codes generated at enrollment. App Users see MFA challenge after password login when factor is enrolled.

## Acceptance criteria

- [ ] `auth.mfa.enroll({ factorType: 'totp' })` returns QR + secret
- [ ] `auth.mfa.challenge(...)` + `auth.mfa.verify(...)` complete enroll
- [ ] Login after enroll requires TOTP challenge
- [ ] Recovery codes issued at enroll; usable once each
- [ ] `auth.mfa.unenroll(...)` removes factor

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
