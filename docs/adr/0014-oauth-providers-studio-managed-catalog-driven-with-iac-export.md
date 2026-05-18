# OAuth providers: Studio-managed with catalog-driven UI, IaC export, OIDC escape-hatch, test-sign-in validation

OAuth provider configuration in NTTS is Studio-managed and DB-stored. Operator inputs flow through Studio UI into `_ntts.auth_oauth_providers`; client_secrets are pgcrypto-encrypted (same class as studio_secrets per ADR-0008) and never live in plain `.env` files. A wrapper layer renders GoTrue's runtime environment from these rows on container start and re-renders on rotation. The Studio UI is driven by a versioned provider catalog manifest (one JSON per NTTS release) so adding a built-in provider does not require frontend code changes. Operators can clone the generic-OIDC adapter to define custom IdPs (e.g. internal Okta/Keycloak variants) without waiting for an NTTS release; non-OIDC/non-SAML providers cannot be operator-added because GoTrue has no generic OAuth2 adapter. Configurations can be exported to a checked-in `auth.oauth.toml` (secrets externalised as `${SECRET_FROM_ENV}` references) for GitOps workflows. Validation is static (schema + OIDC discovery ping) plus a test-sign-in flow that runs the real OAuth dance against a side-effect-free callback. Edit and reveal are admin-only with MFA re-auth.

## Considered Options

- **Pure env-vars (Supabase style).** Rejected: regression vs ADR-0007/0008 secret model; `.env` with 23 client_secrets is a meaningful leak surface; secrets visible to anyone with docker access.
- **Pure declarative file IaC.** Rejected as the only path: hostile to the typical operator workflow ("flip a checkbox" → git commit). Useful as a secondary path.
- **Hardcoded UI per provider.** Rejected: linear-growth maintenance treadmill; every new GoTrue provider needs a frontend release.
- **Operator-defined arbitrary new providers (D-prime).** Rejected at the level of "any OAuth2 spec"; accepted only as cloned variants of generic_oidc/saml. GoTrue's adapter set is closed for non-OIDC.
- **Live healthcheck polling against provider endpoints.** Rejected: low signal value, smells like abuse traffic to providers, doesn't substitute for the real OAuth dance.

## Mechanism

### Storage and injection
- **Table:** `_ntts.auth_oauth_providers (id text PK = provider_id, kind enum('oauth2','oidc','saml','custom_oidc','custom_saml'), enabled bool, fields jsonb (per-catalog-schema, secrets pgcrypto-encrypted with `NTTS_MASTER_KEY`), secret_version int, scopes text[], redirect_url text, last_tested_at timestamptz, updated_at, updated_by)`.
- **Injection.** Studio-backend renders `/var/run/ntts/gotrue.env` (tmpfs) on boot and on `NOTIFY ntts_oauth_change`. GoTrue loads on start; on rotation, GoTrue is sent SIGHUP if it supports config reload, otherwise `docker restart gotrue` (downtime ≤ 5s for new logins; existing sessions are NTTS-issued JWTs and remain valid).

### Provider catalog
- **Source.** JSON manifest shipped in the NTTS image at `/etc/ntts/oauth-providers.catalog.json`, version-bumped per NTTS release.
- **Schema (per entry):** `id, display_name, kind, fields[{name, type:string|secret|url|textarea, required, label, help_text, validate_regex?}], default_scopes, redirect_url_template, setup_guide_url, gotrue_env_mapping`.
- **Rendering.** Studio frontend fetches via `/studio/api/auth/providers/catalog` and renders forms dynamically from schema.
- **Operator custom entries.** Stored in `_ntts.auth_oauth_provider_custom (id, base_kind enum('generic_oidc','saml'), display_name, default_scopes, help_text, setup_guide_url)`. Studio merges custom entries into the catalog at fetch time. Custom entries' configurations use the same `_ntts.auth_oauth_providers` table with `kind='custom_oidc'`/`'custom_saml'`.

### Validation
- **Static.** Server-side schema validation per catalog field types; required-checks; URL/PEM/JSON parse; OIDC `discovery_url` HEAD/GET (must return valid OIDC discovery JSON).
- **Test-sign-in.** `POST /studio/api/auth/oauth/test-flow/:provider_id` (admin + MFA re-auth) returns authorize URL with `state=ntts_test_<random>` and a special callback `…/auth/oauth/test-callback`. Operator completes provider sign-in. Test-callback verifies token-exchange, extracts `sub` + `email`, returns success page. No `auth.users` INSERT, no session cookie. Result lands in `_ntts.auth_oauth_test_log (provider_id, tested_by, tested_at, success, error_message)`.
- **Enable gate.** Soft-warn (not hard-block) if `last_tested_at` is older than 30 days (configurable). UI shows "Last test passed YYYY-MM-DD by user@…" or "✗ failed: invalid_client".

### Rotation
- Two-field UI ("current" + "new"); save bumps `secret_version`, writes audit row, fires NOTIFY, triggers GoTrue reload.
- Active NTTS-session JWTs remain valid. Only new OAuth logins use the new secret. GoTrue's refresh-token handling is internal and not in scope here.

### Role gates
- **Edit, Test, Disable, Delete:** admin only.
- **Reveal client_secret:** admin + MFA re-auth (matches F-FN-10).
- **View provider list (without secrets):** all Operators (`developer` and `viewer` need to know what's available for User-facing flows).

### Disable vs Delete
- **Disable:** sets `enabled=false`, keeps row. Re-enable is one click. GoTrue env drops the `_ENABLED=true` flag.
- **Delete:** separate destructive action (admin + MFA + name-confirm). Wipes row. Audit row captures pre-delete config snapshot with secrets redacted. Affected Users must use a different login method.

### IaC export/import
- `ntts auth oauth export > auth.oauth.toml` produces a config file with secret values replaced by `${SECRET_FROM_ENV:NTTS_OAUTH_<PROVIDER>_SECRET}` placeholders. Safe to commit.
- `ntts auth oauth apply -f auth.oauth.toml` reads file + resolves env-refs + UPSERTs into `_ntts.auth_oauth_providers`. Diff displayed before apply.

### Audit
- **Write actions** (create/update/disable/delete): `_ntts.studio_audit_write` row, `payload={provider_id, action, diff (secrets redacted)}`.
- **Reveal:** `_ntts.studio_audit_read` row.
- **Test results:** `_ntts.auth_oauth_test_log`.
- **GoTrue restart events:** `_ntts.function_audit` (existing system-event home).

## Consequences

- A new table `_ntts.auth_oauth_providers` plus low-volume satellites (`auth_oauth_provider_custom`, `auth_oauth_test_log`) join the `_ntts` schema. Migrations are part of `_ntts` internal migrations.
- The provider catalog is a versioned artefact. Adding a new built-in provider = bumping `oauth-providers.catalog.json` in an NTTS release; rarely needs frontend code changes.
- IaC export is one-way clean (secrets externalised); import is one-way clean (env-refs resolved at apply time). Round-tripping a config through git does not leak secrets.
- Custom OIDC/SAML lets Operators onboard nearly any modern IdP without an NTTS release. Truly proprietary OAuth2 quirks remain blocked on a GoTrue release.
- Test-sign-in callback is the only NTTS endpoint that completes an OAuth token-exchange without producing an `auth.users` row. It is admin-MFA-gated and audit-logged, but it is a sensitive surface and must be implemented with strict state-token verification and short-lived (5 min) test session.
- Rotation has a brief (≤5s) window of new-login failures during GoTrue reload. Documented; sessions stay valid; not a meaningful availability issue.
