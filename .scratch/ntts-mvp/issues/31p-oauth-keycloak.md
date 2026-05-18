Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Keycloak (self-hosted OIDC) as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest with `client_id`, `client_secret`, `base_url`, `realm`, scopes (default `openid email profile`), callback. `gotrue_env_mapping` for GoTrue's Keycloak provider.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'keycloak' })` against Operator-configured base_url + realm issues JWT
- [ ] Studio form renders fields for base_url + realm
- [ ] OIDC discovery against constructed URL succeeds at save
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
