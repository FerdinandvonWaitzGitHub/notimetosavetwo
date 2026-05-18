Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Wire the **generic OIDC** adapter as a built-in OAuth provider via the catalog driver (slice 30). Catalog manifest with `client_id`, `client_secret`, `issuer_url`, scopes (default `openid email profile`), callback, optional `display_name` override. Operator can clone this adapter for arbitrary OIDC IdPs (per F-AUTH-10 custom-provider clause). Static validation runs OIDC discovery against the entered issuer.

## Acceptance criteria

- [ ] `signInWithOAuth({ provider: 'oidc:<clone-name>' })` works against any conformant OIDC IdP
- [ ] Studio renders fields for issuer URL + display name + scopes
- [ ] OIDC discovery against issuer succeeds at save (well-known + jwks)
- [ ] Clone-from-generic flow creates a new catalog entry with Operator-chosen `display_name`
- [ ] Test-Sign-In completes against the configured IdP; result in `_ntts.auth_oauth_test_log`
- [ ] Disable + Delete + Rotation + IaC export round-trip
- [ ] Cannot define arbitrary new OAuth2 providers (GoTrue has no generic OAuth2 adapter; F-AUTH-10)

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
