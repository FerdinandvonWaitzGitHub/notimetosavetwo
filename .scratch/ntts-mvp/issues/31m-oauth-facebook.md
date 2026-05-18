Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Facebook as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest with `client_id` (App ID), `client_secret` (App Secret), scopes (default `email`), callback. `gotrue_env_mapping` for GoTrue's Facebook provider.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'facebook' })` issues JWT
- [ ] Studio form renders from catalog manifest
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
