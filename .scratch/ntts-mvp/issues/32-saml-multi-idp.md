Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

SAML 2.0 via GoTrue SSO config; **multi-IdP** per Deployment with email-domain-affinity-routing (F-AUTH-9). Studio-UI lists all configured IdPs in `_ntts.auth_saml_providers (id, metadata_xml_or_url, email_domain_pattern, display_name, priority, enabled_at)`. Login surface shows initial email-only input; after entry, GoTrue routes to the IdP whose `email_domain_pattern` matches. Fallback on multiple matches: Operator priority field. GoTrue fork-patch required (per ADR-0023 Addendum). CLI: `ntts auth saml add/list/test/disable/delete`.

## Acceptance criteria

- [ ] Two SAML IdPs configured with different `email_domain_pattern` route correctly by entered email
- [ ] Operator priority resolves ambiguity when multiple patterns match
- [ ] `ntts auth saml add --metadata-url https://...` adds an IdP
- [ ] `ntts auth saml test <id>` issues a real SP-init SAML auth request and validates the response
- [ ] Disable preserves row + drops enabled flag; Delete requires MFA + name-confirm + audit
- [ ] Studio-UI lists all configured IdPs with status

## Blocked by

- [08-app-user-auth-core.md](./08-app-user-auth-core.md)
