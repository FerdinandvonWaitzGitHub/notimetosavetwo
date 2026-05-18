# Three classes of secrets, separated by consumer

NTTS distinguishes three secret classes, each with a different store, consumer, master-key, and rotation path. No unified "secrets manager" — the abstraction would obscure the trust boundaries.

| Class | Store | Consumer | Master key | Rotation |
|---|---|---|---|---|
| **System-Bootstrap** | Container env (operator `.env`) | Gateway, GoTrue, Edge Runtime, Postgres | n/a | compose-restart + runbook |
| **Function Secret** | `_ntts.function_secrets` (pgcrypto) | Edge Runtime user-workers, via `Deno.env.get(name)` | `NTTS_MASTER_KEY` (Edge Runtime env) | NOTIFY + worker eviction ([ADR-0007](./0007-secret-injection-worker-boot-rotation-via-notify.md)) |
| **Vault Secret** | `vault.secrets` (pgsodium) | SQL: PL/pgSQL, FDW (`wrappers`), pg_net, RLS policies | pgsodium server key (Postgres data dir) | `vault.update_secret(...)` |

## Considered Options

- **Single unified secrets store.** Rejected: collapses three different trust boundaries (env, edge-runtime, sql-tier) into one, makes consumer-specific encryption guarantees impossible to reason about.
- **Function Secrets in Vault.** Rejected: Edge Runtime would need a SECURITY DEFINER round-trip per worker boot to fetch them, and pgsodium decrypt-on-read is heavier than the per-worker-boot pgcrypto path.
- **Three classes (chosen).** One store per consumer, one master key per store, no cross-contamination.

## Consequences

- Operators who need the same value in both an Edge Function and a SQL FDW store it twice. Studio surfaces the relationship ("linked to vault entry") and rotates both with one click.
- The AI provider key (see [ADR-0003](./0003-ai-assistant-provider-agnostic-default-off.md)) lives in `vault.secrets[ai_provider_key]` because the consumer is Studio-backend SQL, not a user Function.
- Vault decrypt calls must not land in `pg_stat_statements` parameter logs; access via `SECURITY DEFINER` wrappers, not direct `SELECT` from `vault.decrypted_secrets`.
- Backup runbook must call out **both** master keys separately. Losing either means the encrypted content in that store is dead weight after restore.
- Studio UI has **two** secrets surfaces: "Function Secrets" (per Function) and "Vault". Not one. The separation is the feature, not the bug.
