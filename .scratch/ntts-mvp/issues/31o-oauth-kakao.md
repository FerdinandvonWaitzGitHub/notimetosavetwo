Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Kakao as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest with `client_id` (REST API Key), `client_secret` (optional), scopes (default `account_email profile_nickname`), callback. `gotrue_env_mapping` for GoTrue's Kakao provider.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'kakao' })` issues JWT
- [ ] Studio form renders from catalog manifest (incl. optional `client_secret`)
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
