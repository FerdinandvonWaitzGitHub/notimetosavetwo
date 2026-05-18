Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Cmd+G opens an inline glossary palette in Studio. Source-of-truth is `CONTEXT.md` (the canonical NTTS vocabulary). Glossary entries surface the term, the definition, the `_Avoid_:` notes, and links to relevant ADRs. Build-time step parses `CONTEXT.md` and emits a JSON catalog that ships with Studio Frontend; rebuild step gates on CI to keep glossary fresh.

## Acceptance criteria

- [ ] Cmd+G opens glossary palette from any Studio page
- [ ] Search filters across term + definition; selecting a term scrolls to full entry
- [ ] All headwords from CONTEXT.md "Language" section present (Operator, Studio User, App User, Role, Setup Token, Deployment, Cluster Mode, Studio-Backend, Control Plane, ntts-typegen, ntts-egress-proxy, Gateway, Trust Boundary, Service Role, Function, Bundle, Version, Visibility, Function Secret, Master Key, Vault Secret, System-Bootstrap Secret, Pre-deploy Pipeline, Drop-in for Supabase)
- [ ] `_Avoid_:` notes rendered as warning callouts
- [ ] ADR links rendered as clickable
- [ ] CI fails build if Studio glossary catalog drifts from CONTEXT.md headwords

## Blocked by

- [07a-studio-shell-login-mfa-nav.md](./07a-studio-shell-login-mfa-nav.md)
