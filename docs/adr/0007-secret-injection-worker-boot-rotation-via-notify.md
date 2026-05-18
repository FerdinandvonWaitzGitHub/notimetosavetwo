# Secret injection: at worker boot, rotation via NOTIFY + eviction

Function secrets are not bundled. At Edge Runtime user-worker boot, secrets are read from `_ntts.function_secrets`, decrypted with the master key from container env (`NTTS_MASTER_KEY`), and set as worker-level env. Functions read them via `Deno.env.get('NAME')` (Supabase-compatible) — `process.env.NAME` works via the Deno→Node compat shim. Worker env is stable for the worker lifetime. Secret rotation by the Operator triggers `NOTIFY ntts_fn_secret_change`; the Edge Runtime evicts affected workers and the next invocation spins a fresh worker with the new secret.

## Considered Options

- **JIT-per-invocation env injection.** Rejected: V8 isolate env is per-isolate, not per-invocation; would require fresh isolate per call, killing warm-isolate performance.
- **Argument-passed secrets via `ctx.secrets.X`.** Rejected: breaks Supabase drop-in compat — existing functions use `Deno.env.get(...)`.
- **At-worker-boot env (chosen).** Aligns with how Edge Runtime actually works; rotation latency is bounded by NOTIFY-driven eviction (~<1s in normal operation).

## Consequences

- Newly rotated secrets are visible to invocations within a sub-second window after rotation, not literally synchronously. Documented in Studio UI and runbook.
- `NTTS_MASTER_KEY` is the lynchpin: lose it, lose all encrypted secrets. Backup runbook makes this explicit. Single-tenant trust model accepts that an Operator-with-shell has access to all decrypted secrets — same blast-radius assumption as the rest of the appliance.
- Secrets are per-Function (`function_id` as primary key dimension), not global. If the same value is needed in N Functions, it's stored N times. Studio offers a "copy from another Function" affordance.
- Studio "Reveal Secret" requires MFA re-auth and lands a row in `_ntts.studio_audit_read`.
- The Vault feature (pgsodium-backed) is a separate, stronger optional layer (see [ADR-0008] when written) for secrets that should survive `NTTS_MASTER_KEY` compromise.
