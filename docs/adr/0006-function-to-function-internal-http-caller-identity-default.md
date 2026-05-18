# Function-to-Function: internal HTTP, caller identity by default, explicit elevation

Function-to-Function calls go through the **same internal HTTP path as event/cron triggers**, hitting an Edge Runtime endpoint on the internal Docker network. Each call spins (or re-uses) a fresh user-worker isolate — same crash-isolation, same quotas, same log pipeline as a public HTTP invocation. The caller's identity is propagated by default; elevation to `service_role` requires an explicit `as: 'service'` option and is recorded in the Bundle manifest.

## Considered Options

- **In-process direct call.** Rejected: collapses isolate boundary between A and B, breaks quota accounting and crash isolation.
- **Postgres-queue indirection.** Rejected: async-only, no return value, unsuitable for the synchronous RPC pattern Functions need.
- **Default service-role elevation.** Rejected: silently elevates privileges, makes RLS holes invisible.
- **Internal HTTP + caller identity by default (chosen).** One code path for public HTTP, f2f, event, cron. Caller identity is propagated, elevation is opt-in and audited.

## Consequences

- A new SDK method `ntts.functions.invoke(name, payload, { as?: 'caller' | 'service' })` becomes part of the public contract.
- The Bundle manifest records each f2f target with its `as` choice; Pipeline can warn on repeated `service_role` elevation.
- Auth hooks, cron, and pg_net DB-event triggers all reduce to "call the internal endpoint with service-JWT" — uniform path.
- Recursion depth limited to 5 via `X-NTTS-Call-Depth` header; cycles caught at Bundle time via manifest closure walk, not at runtime.
- The internal `/_internal/invoke/:name` endpoint is **only reachable from the internal Docker network** (asserted in F-GW-8) and additionally HMAC-signed with a key from the Edge Runtime env.
