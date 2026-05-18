Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Twitter/X (OAuth 2.0 with PKCE) as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest with `client_id`, `client_secret`, scopes (default `users.read tweet.read offline.access`), callback. `gotrue_env_mapping` for GoTrue's Twitter provider.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'twitter' })` issues JWT
- [ ] PKCE flow handled by GoTrue
- [ ] Studio form renders from catalog manifest
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
