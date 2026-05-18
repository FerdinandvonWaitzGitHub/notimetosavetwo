Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

JS SDK `@ntts/client` — thin re-export of `@supabase/supabase-js` plus NTTS-generated type augmentation (F-SDK-1..3, F-SDK-5). Type-augmentation reads the schema fingerprint from `_ntts.generated_types` to give Operators full row-level typing in their app code. Distributed via npm; versioned lockstep with NTTS minor (1.3.x ↔ 1.3.x), patch versions independent for SDK-only bugfixes.

## Acceptance criteria

- [ ] `npm install @ntts/client` installs latest
- [ ] `createClient(NTTS_URL, NTTS_ANON_KEY)` matches `@supabase/supabase-js` shape (§6 #1, F-SDK-1)
- [ ] TS generics from schema work via generated types
- [ ] Existing `@supabase/supabase-js` app runs unchanged against NTTS (§6 #1)
- [ ] `.channel().on('postgres_changes', ...)` works (§6 #13)
- [ ] `.rpc('fn_name', { ... })` works (§6 #14)
- [ ] GraphQL works via `urql`/`graphql-request` (§6 #15)
- [ ] Lockstep versioning: every NTTS minor release publishes a matching `@ntts/client` version

## Blocked by

- [06-type-gen-worker.md](./06-type-gen-worker.md)
