Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Magic-link sign-in (F-AUTH-3): 15-min single-use link via email. supabase-js `auth.signInWithOtp({ email })` triggers the email; clicking the link issues a JWT. Email body templated via Go-text/template in `_ntts.auth_templates` (per-locale).

## Acceptance criteria

- [ ] `auth.signInWithOtp({ email })` sends a magic-link email
- [ ] Clicking the link within 15 min returns a JWT
- [ ] Re-clicking the same link rejects (single-use)
- [ ] Link expiry tested via clock advance (>15 min → rejected)
- [ ] Email body honours per-locale template (DE default)

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
