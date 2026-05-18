Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Vault (pgsodium-backed) for SQL-tier consumers per ADR-0008 (F-VLT-1..5). `vault.create_secret(value, name, description)` + `vault.update_secret(name, value)` available to `admin` Operator via Studio. Decryption only via `SECURITY DEFINER` wrapper functions; direct `SELECT * FROM vault.decrypted_secrets` requires `service_role`. Vault decrypt calls excluded from `pg_stat_statements` parameter capture. AI-provider key (F-VLT-4) stored as `vault.secrets[ai_provider_key]`. Function code (TS in Edge Runtime) **does not** read Vault — use Function Secrets (slice 14) instead. Separate UI tab in Studio (clearly distinct from Function Secrets).

## Acceptance criteria

- [ ] `vault.create_secret('value', 'name', 'description')` writes encrypted row
- [ ] PL/pgSQL function consuming the secret via SECURITY DEFINER wrapper returns correct value
- [ ] `SELECT * FROM vault.decrypted_secrets` as `nss_admin` → permission denied; as `service_role` → allowed
- [ ] `pg_stat_statements` does not capture plaintext secret values
- [ ] Studio Vault tab separate from Function Secrets tab; copy makes the distinction clear
- [ ] CLI `ntts vault set/ls/rotate` works; admin-only

## Blocked by

- [03-postgres-roles-three-tier.md](./03-postgres-roles-three-tier.md)
