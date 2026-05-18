# Postgres superuser: three-tier access, `nss_admin` is the default, true superuser is emergency-only

The Postgres `postgres` superuser is bound to the Unix socket inside the container, has a random 32-byte password generated at first `up` and stored in `_ntts.platform_secrets[postgres_superuser]` (pgcrypto-encrypted with `NTTS_MASTER_KEY`), and is not reachable through any normal NTTS path. Operator SQL work runs as `nss_admin`, a non-superuser role with elevated grants but explicit caps (no SUPERUSER, no REPLICATION, no `ALTER USER postgres`). True superuser access is reachable only through `ntts admin shell --superuser` — MFA re-auth + a written reason + a 30-second cool-down — and every statement is audit-logged with full text and reason. The result: 95% of Operator SQL flows through the audited `nss_admin` path; the dangerous path exists, is reachable, but is loud, slow, and traceable.

## Considered Options

- **Operator-as-superuser by default.** Rejected: superuser bypasses RLS and audit triggers; treating it as routine punctures every other security control in this PRD.
- **No NTTS-specific role; rely on standard Postgres roles.** Rejected: standard Postgres roles either over- or under-grant. `nss_admin` is the sized-right "Operator-routine" role that does not exist out-of-box.
- **Operator never gets superuser (true lockout).** Rejected: emergency recovery scenarios (corruption, replication-stream repair, extension upgrades that need SUPERUSER) require a path. Better to make the path expensive than to remove it.
- **Operator-supplied superuser password as the only bootstrap option.** Rejected: most Operators just want `ntts up` to work. Random-gen-with-env-override (`NTTS_BOOTSTRAP_POSTGRES_PASSWORD`) covers both.

## Mechanism

### Role layout
- **`postgres`** — true SUPERUSER. Unix-socket-only (`listen_addresses` does not advertise `postgres`-role TCP). Password random-generated at first boot, stored in `_ntts.platform_secrets[postgres_superuser]` (pgcrypto with `NTTS_MASTER_KEY`).
- **`nss_admin`** (NTTS-owned) — `NOSUPERUSER`, `BYPASSRLS`, grants:
  - `ALL ON SCHEMA _ntts, public, auth, storage, realtime, vault, extensions` (or as service-schemas evolve).
  - `CREATE` on database, `ALTER` on owned objects.
  - Explicitly **denied**: `REPLICATION`, `ALTER USER postgres`, `CREATEROLE` for superuser-bit, access to `pg_authid.rolpassword`.
- **`supabase_auth_admin`, `supabase_storage_admin`, `supabase_realtime_admin`, `supabase_functions_admin`** — standard Supabase-service owners, grants scoped to the corresponding service schema.
- **`anon`, `authenticated`, `service_role`** — app-tier roles, standard Supabase JWT-claim mapping.
- **Standard pool/internals** — `pgbouncer`, `supabase_admin_dba`, etc., as upstream.

### Access tiers
1. **Studio SQL Editor (routine).** Connection runs as `nss_admin`. Each statement audited to `_ntts.studio_audit_write` with `payload={action:'sql_execute', statement_hash, statement_text:truncate(1000), affected_rows, duration_ms}`. PII redaction is best-effort against statement text (named patterns).
2. **CLI shell (scripts).** `ntts admin sql --file <file>` or `ntts admin shell` (interactive). MFA prompt. Connects through `ntts-edge` as `nss_admin`. Statements pass through the same audit pipeline as Studio SQL editor.
3. **Superuser shell (emergency).** `ntts admin shell --superuser`. Sequence: MFA re-auth → confirmation prompt with a non-empty `Reason` field → 30-second cool-down before psql opens. CLI fetches `postgres` password from `platform_secrets` via the Vault-reveal pattern. Opens psql through `docker exec` (Unix socket inside the container) — explicitly not through `ntts-edge` because `postgres` does not accept TCP. Every statement audit-logged with `payload={action:'superuser_sql', session_id, reason, statement_text}`. Banner notification fanned out to all active admin Studio sessions: "Superuser session active by user@… for reason …". Session also written as a top-level event to `_ntts.studio_audit_write` at open and close.

### Bootstrap
- First `up` with empty `platform_secrets[postgres_superuser]`:
  - If `NTTS_BOOTSTRAP_POSTGRES_PASSWORD` is set: use it, INSERT-encrypt into platform_secrets, clear from process env after migration.
  - Otherwise: generate 32-byte random, INSERT-encrypt, never display.
- Pre-boot health-check refuses to start if `platform_secrets[postgres_superuser]` is unreadable (master-key mismatch or row missing) → clear Operator message pointing to recovery doc.

### Audit visibility on the OS side
- `pgaudit` extension (already shipped in §4.1) logs every SUPERUSER login at the Postgres server-log layer.
- Container stdout/stderr → host docker logs; F-LOG infrastructure does not duplicate this, but operator-managed external log shipping picks it up.
- A future `_ntts.system_logs` table could ingest pgaudit lines for in-NTTS visibility; out of v1.0 scope.

### Role gates
- Creating new application-level Postgres roles: `nss_admin` allowed via Studio "Roles" UI; routes through audit.
- Editing `postgres` superuser password: only `ntts admin rotate-superuser-password` (MFA + reason + cool-down); rotates `platform_secrets` row + restarts dependent containers that hold cached password (none in normal NTTS — all containers connect as service-specific roles).

## Consequences

- A new role (`nss_admin`) and a new table (`_ntts.platform_secrets`) are part of `_ntts` schema; both included in `pg_dump`.
- Container-internal Unix-socket binding for `postgres` requires the Postgres container to be configured to advertise the socket on the right path and to allow `local` peer auth for `postgres` only. Standard Postgres config; not exotic.
- Emergency superuser path is expensive on purpose: MFA + reason + cool-down + audit banner. Operators who run automated tooling against the superuser path will find it deliberately hostile — this is the design intent.
- The `nss_admin` grants matrix is sizable and version-bound to NTTS schema. Migration `_ntts` versions that add new service schemas must extend `nss_admin` grants in the same migration.
- The audit text-truncation (1000 chars) loses detail on very long statements; tradeoff accepted to keep audit table size reasonable. Operators wanting full text use external Postgres query logging.
- Banner notifications during superuser session require a live channel from `nss_admin`-write to all open Studio sessions — implemented via `NOTIFY ntts_admin_event` which Studio listens to. Adds one more NOTIFY channel.
