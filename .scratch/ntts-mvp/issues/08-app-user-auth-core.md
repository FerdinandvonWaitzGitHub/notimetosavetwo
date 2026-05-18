Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

GoTrue wrapped behind Gateway with JWT-verify disabled (GoTrue trusts internal headers per F-GW-2). Gateway validates JWT once at the boundary and sets `X-NTTS-User-Id`, `X-NTTS-Role`, `X-NTTS-Request-Id` internal headers (F-GW-3, never accepted from external clients). Endpoints: signup, login, refresh, logout, password reset. JWT claims `aud`, `exp`, `sub`, `email`, `role`, `app_metadata`, `user_metadata` per F-AUTH-1. Custom SMTP via env-vars; password-reset emails blocked until SMTP configured (proper template work lands in slice 38). Request-ID propagated to upstream calls and into `_ntts.function_logs.request_id`. Rate-limit (sliding-window per route × identity) state in `_ntts.rate_limit_buckets` per F-GW-6.

## Acceptance criteria

- [ ] `@supabase/supabase-js` `createClient().auth.signUp({ email, password })` returns a JWT (§6 #1 partial)
- [ ] `auth.signInWithPassword()` returns a JWT; refresh + logout work
- [ ] JWT claims include `aud`, `exp`, `sub`, `email`, `role`, `app_metadata`, `user_metadata`
- [ ] Gateway sets internal headers; PostgREST/GoTrue/Storage/Realtime/Edge Runtime have JWT-verify disabled
- [ ] External request attempting to set `X-NTTS-*` headers gets them stripped
- [ ] Request-ID (UUIDv7) propagated to all upstreams
- [ ] Default rate-limit: 60 RPM per IP for unauthenticated, 600 RPM per App User authenticated, no limit for service_role
- [ ] Password-reset returns explanatory error until SMTP configured

## Blocked by

- [01-compose-greens-up.md](./01-compose-greens-up.md)
