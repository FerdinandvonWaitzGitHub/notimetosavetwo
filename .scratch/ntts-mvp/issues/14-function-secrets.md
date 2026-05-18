Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Function Secrets per ADR-0007: per-Function named pgcrypto-encrypted values in `_ntts.function_secrets`, master-key `NTTS_MASTER_KEY` in Edge Runtime container env only. Read at user-worker boot (not per-invocation); worker env stable for worker lifetime. API in Function code: `Deno.env.get('NAME')` primary, `process.env.NAME` via Deno compat shim. Never bundled into eszip; manifest may reference secret *names*. Rotation: Operator changes a secret → `NOTIFY ntts_fn_secret_change, '{function_id, name}'` → Edge Runtime evicts affected workers → next invocation sees new value (sub-second propagation). Log pipeline replaces plaintext matches of secret values → `[REDACTED:NAME]`. Studio Reveal: admin-only, MFA re-auth required, lands `_ntts.studio_audit_read` row. CLI: `ntts fn secrets set/ls/rotate/reveal`.

## Acceptance criteria

- [ ] `ntts fn secrets set FOO bar` creates encrypted row in `_ntts.function_secrets`
- [ ] Function code reads `Deno.env.get('FOO')` and returns "bar"
- [ ] `console.log(SECRET)` log shows `[REDACTED:FOO]` (§6 #8)
- [ ] Secret rotation propagates in <1s via NOTIFY + worker eviction
- [ ] Bundle eszip does not contain plaintext secret values
- [ ] `ntts fn secrets reveal FOO` requires admin + MFA re-auth; lands `_ntts.studio_audit_read` row
- [ ] Same secret value used in N Functions is stored N times (per-Function scoping)

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
