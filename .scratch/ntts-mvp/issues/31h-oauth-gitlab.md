Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire GitLab (gitlab.com + self-hosted) as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest with `client_id`, `client_secret`, `base_url` (default `https://gitlab.com`), scopes (default `read_user openid`), callback. `gotrue_env_mapping` for GoTrue's GitLab provider with custom base-URL support.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'gitlab' })` issues JWT against gitlab.com
- [ ] Self-hosted base URL configurable and validates against schema
- [ ] OIDC discovery succeeds for entered base URL at save
- [ ] Test-Sign-In completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
