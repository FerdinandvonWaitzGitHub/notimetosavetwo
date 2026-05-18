Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Azure AD (work/school tenant explicit) as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest with `client_id`, `client_secret`, `tenant` (org tenant ID required), scopes (default `openid email profile`), callback. Differs from slice 31e Microsoft consumer flow by requiring explicit `tenant` and rejecting `common`/`consumers`. `gotrue_env_mapping` for GoTrue's Azure provider in tenant-specific mode.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'azure' })` (work tenant) issues JWT
- [ ] Studio form rejects `common`/`consumers` as `tenant` value
- [ ] OIDC discovery against `https://login.microsoftonline.com/<tenant>/v2.0` succeeds at save
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
