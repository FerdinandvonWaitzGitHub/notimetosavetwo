Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

RPC via PostgREST native `/rest/v1/rpc/<fn>`. Postgres functions in `public.*` exposed automatically; respect RLS via `SECURITY INVOKER` (default) or bypass via `SECURITY DEFINER` (explicit Operator choice). Supabase-js `.rpc('fn_name', { … })` works unchanged. Type-gen (slice 6) surfaces RPC signatures in the generated types.

## Acceptance criteria

- [ ] `.rpc('search', { q: 'foo' })` returns expected rows via supabase-js (§6 #14)
- [ ] Generated types include RPC function signatures
- [ ] RLS respected on `SECURITY INVOKER` RPCs
- [ ] `SECURITY DEFINER` RPCs marked clearly in generated types
- [ ] Error body matches PostgREST native format

## Blocked by

- [09-rest-rls.md](./09-rest-rls.md)
