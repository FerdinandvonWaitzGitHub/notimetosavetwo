Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Slack as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest entry with `client_id`, `client_secret`, scopes (default `openid email profile`), callback. `gotrue_env_mapping` for GoTrue's Slack OIDC provider.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'slack' })` issues JWT
- [ ] Studio form renders from catalog manifest
- [ ] OIDC discovery succeeds at save
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
