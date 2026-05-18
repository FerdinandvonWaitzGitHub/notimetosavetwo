Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Function-to-Function call per ADR-0006. SDK in Function code: `await ntts.functions.invoke(name, payload, { as?: 'caller' | 'service' })`. Default `as: 'caller'`. Transport = internal HTTP to `http://edge-runtime:9000/_internal/invoke/:name`, HMAC-signed with `hmac_f2f` from Edge Runtime env. Endpoint not reachable through Gateway. Identity propagation: caller's `X-NTTS-User-Id` + `X-NTTS-Role` forwarded by default; `as: 'service'` replaces with service-role internals. Recursion depth ≤5 via `X-NTTS-Call-Depth`; cycles caught at Bundle time (slice 15). Typed failure modes: `FunctionNotFound`, `FunctionTimeout`, `FunctionQuotaExceeded`, `FunctionRecursionLimit`.

## Acceptance criteria

- [ ] `ntts.functions.invoke('private-fn')` from public Function succeeds (default `as: 'caller'`)
- [ ] Caller's user-id + role propagated; private Function's RLS sees the App User context
- [ ] `as: 'service'` replaces identity with `service_role`
- [ ] Depth ≥6 throws `FunctionRecursionLimit`
- [ ] Calling missing Function → `FunctionNotFound`
- [ ] Failing HMAC sig → 401 (no execution, no log row)
- [ ] Endpoint not reachable from outside the Docker network

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
