Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Microsoft (consumer + multi-tenant work/school) as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest entry with `client_id`, `client_secret`, `tenant` selector (common/organizations/consumers/<id>), scopes (default `openid email profile`), callback. `gotrue_env_mapping` for GoTrue's Azure provider in Microsoft-consumer mode.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'azure' })` (consumer-tenant) issues JWT
- [ ] Studio form renders tenant selector
- [ ] OIDC discovery succeeds against chosen tenant at save
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
