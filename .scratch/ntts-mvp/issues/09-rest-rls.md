Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

PostgREST wrapped at `/rest/v1` with JWT-verify off via config; Gateway sets internal identity headers. Full PostgREST filter DSL (incl. `or=(a.eq.1,b.eq.2)`), FK-embed `select`, bulk + `Prefer: return=representation`, Range-header pagination, PostgREST error body format (`code`, `message`, `details`, `hint`). RLS enforced on all paths. Service-role bypass only on service-JWT (F-GW-5 detects `role: service_role`, validated against secret known only to Gateway + Operator env). Demoable via supabase-js `from('table').select()` returning only RLS-allowed rows.

## Acceptance criteria

- [ ] `@supabase/supabase-js` `from('table').select()` returns RLS-allowed rows for authenticated App User (§6 #1)
- [ ] Full filter DSL works incl. `or=(a.eq.1,b.eq.2)`
- [ ] FK-embed `select=*,author(*)` works
- [ ] Bulk insert + `Prefer: return=representation` works
- [ ] Range-header pagination works
- [ ] Error responses match PostgREST native format
- [ ] Service-role JWT bypasses RLS as expected
- [ ] Default-deny verified: table with RLS on, no policies → anon + authenticated both denied (ADR-0024)

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
