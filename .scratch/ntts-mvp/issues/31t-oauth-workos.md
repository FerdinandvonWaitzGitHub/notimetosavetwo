Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire WorkOS as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest with `client_id`, `api_key`, `connection_id` (optional), scopes (default `openid email profile`), callback. `gotrue_env_mapping` for GoTrue's WorkOS provider.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'workos' })` issues JWT
- [ ] Studio form renders WorkOS-specific fields (api_key + connection_id)
- [ ] OIDC discovery succeeds at save
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
