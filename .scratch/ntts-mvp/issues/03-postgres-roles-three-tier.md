Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Three-tier Postgres access model per ADR-0018. `postgres` SUPERUSER is bound to Unix-socket-only inside the Postgres container, random-password generated at first `up`, stored in `_ntts.platform_secrets[postgres_superuser]` (pgcrypto with `NTTS_MASTER_KEY`). `nss_admin` role (`NOSUPERUSER`, `BYPASSRLS`, broad `_ntts.*` + `public.*` + service-schema grants, denied `REPLICATION` + `ALTER USER postgres` + superuser-CREATEROLE) is the operator-routine SQL role. Standard Supabase service-owner + app roles (`supabase_auth_admin`, `supabase_storage_admin`, `supabase_realtime_admin`, `supabase_functions_admin`, `anon`, `authenticated`, `service_role`) created from upstream template. Three access tiers: Studio SQL Editor as `nss_admin`; `ntts admin shell/sql` as `nss_admin` through Gateway with audit; `ntts admin shell --superuser` requires MFA + non-empty reason + 30s cool-down via `docker exec` over Unix socket. Default-GRANT matrix from F-DB-7 enforced via Migrations-Linter (refused `GRANT … TO anon|authenticated` on `_ntts.*`, `auth.*`, `storage.*`, `realtime.*`, `vault.*`).

## Acceptance criteria

- [ ] `postgres` superuser unreachable via TCP (Unix-socket-only inside container)
- [ ] `nss_admin` role created with F-DB-2 grants; `REPLICATION`, `ALTER USER postgres`, superuser-CREATEROLE all denied
- [ ] All 7 standard Supabase roles created at first `up`
- [ ] Default-GRANT matrix (F-DB-7) applied; `0002_role_grants.sql` is the canonical source
- [ ] `ntts admin shell --superuser` requires MFA re-auth + non-empty reason + 30s cool-down; every statement audit-logged with reason + session-id
- [ ] Banner notification fanned out to all open admin Studio sessions via `NOTIFY ntts_admin_event` on superuser-shell open
- [ ] `ntts admin rotate-superuser-password` (MFA + reason + cool-down) rotates `postgres` password in platform_secrets
- [ ] Pre-boot health-check refuses startup if `platform_secrets[postgres_superuser]` is unreadable (master-key mismatch or row missing)

## Blocked by

- [01-compose-greens-up.md](./01-compose-greens-up.md)
