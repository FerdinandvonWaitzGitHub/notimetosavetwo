Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire GitHub as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest entry with `client_id`, `client_secret`, scopes (default `user:email`), callback URL. `gotrue_env_mapping` for GoTrue's GitHub provider.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'github' })` issues JWT
- [ ] Studio form renders dynamically from catalog manifest
- [ ] OIDC discovery skip / GitHub-specific endpoint validated at save
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
