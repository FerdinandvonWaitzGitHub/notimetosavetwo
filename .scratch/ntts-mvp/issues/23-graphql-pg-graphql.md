Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

`pg_graphql` extension exposed at `/graphql/v1` via PostgREST routing. RLS-respecting. Introspection enabled in dev, disabled in prod by default. Schema-cache reload piggybacks the `NOTIFY pgrst` from migrations (slice 5). `urql` / `graphql-request` clients work unchanged against the path.

## Acceptance criteria

- [ ] `POST /graphql/v1` with introspection query returns schema in dev mode
- [ ] Authenticated GraphQL query returns RLS-filtered rows (§6 #15)
- [ ] Introspection disabled by default in prod
- [ ] Schema-cache reload after migration is verified before `ntts db migrate` returns
- [ ] `urql` + `graphql-request` smoke tests pass

## Blocked by

- [09-rest-rls.md](./09-rest-rls.md)
