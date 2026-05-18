Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire Apple as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest entry with Apple-specific fields (Client ID, Team ID, Key ID, private key, scopes, callback). Apple uses a private-key-derived client secret (JWT signed with the Apple Auth Key); render the key-upload + auto-JWT-generation in the Studio form. `gotrue_env_mapping` configured for GoTrue's Apple provider.

## Acceptance criteria

- [ ] `supabase-js.auth.signInWithOAuth({ provider: 'apple' })` issues JWT end-to-end
- [ ] Studio form renders Apple-specific fields (Team ID, Key ID, .p8 upload)
- [ ] Catalog manifest entry validates against schema in CI
- [ ] Static OIDC discovery check on save (skip for Apple if non-OIDC config required — use Apple's documented endpoint)
- [ ] Test-Sign-In flow completes; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip work like Google (slice 30)

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
