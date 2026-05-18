Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

OAuth catalog driver per F-AUTH-10 / ADR-0014. Provider catalog as versioned JSON manifest in NTTS image describes each built-in provider's fields, labels, validation, `gotrue_env_mapping`. Studio renders setup forms dynamically. **Storage**: `_ntts.auth_oauth_providers` row per provider; client_secrets pgcrypto-encrypted (studio_secrets class per ADR-0008). **Injection**: studio-backend renders `/var/run/ntts/gotrue.env` on boot and on `NOTIFY ntts_oauth_change`; GoTrue reloads. **Validation**: static (schema + OIDC discovery ping) plus admin-MFA-gated Test-Sign-In flow that completes real OAuth dance against side-effect-free callback; results in `_ntts.auth_oauth_test_log`. **Rotation**: two-field UI, secret-version bump, NOTIFY → GoTrue reload; NTTS-issued sessions stay valid. **Role gates**: edit/reveal/test/disable/delete = admin + MFA re-auth; view-without-secrets = all Operators. **Disable** (default): keep row, drop `_ENABLED=true`. **Delete**: destructive, MFA + name-confirm, audit snapshot. **IaC export/import**: `ntts auth oauth export > auth.oauth.toml` (secrets externalised as refs), `ntts auth oauth apply -f`. Google is the first wired provider as the tracer; remaining providers fan out across slices 31a..31u.

## Acceptance criteria

- [ ] Google OAuth signup via `supabase-js.auth.signInWithOAuth({ provider: 'google' })` issues JWT end-to-end
- [ ] Studio renders Google config form dynamically from catalog manifest
- [ ] Static validation: schema + OIDC discovery ping at save time
- [ ] Admin-MFA-gated Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Secret rotation propagates via NOTIFY + GoTrue reload in <2s
- [ ] Role gates: viewer cannot edit; reveal requires admin + MFA re-auth
- [ ] IaC: `ntts auth oauth export` + `apply -f` round-trip preserves all enabled providers

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
- [07b-studio-audit-write-read-every-page.md](./07b-studio-audit-write-read-every-page.md)
