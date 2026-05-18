Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Anonymous sign-in (F-AUTH-8): supabase-js `auth.signInAnonymously()` returns JWT with `aud: anon`. RLS policies can target `auth.role() = 'anon'`. Upgrade flow: anonymous user can later link an email/phone via `auth.linkIdentity(...)` without losing identity.

## Acceptance criteria

- [ ] `auth.signInAnonymously()` returns JWT with `aud: anon`
- [ ] `auth.users` row exists with no email/phone
- [ ] RLS policy `auth.role() = 'anon'` applies as expected
- [ ] `auth.linkIdentity({ provider: 'email' })` upgrades the user; same `sub` retained
- [ ] Capacity test: 1.000 anonymous sessions sustainable

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
