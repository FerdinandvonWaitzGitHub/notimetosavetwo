Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Studio Frontend skeleton (Next.js, shadcn/ui Components in `apps/studio-frontend/src/ui/`, design tokens in `apps/studio-frontend/src/tokens/`, SF-Pro-Display + Inter fallback, light + dark mode, Apple-HIG + WCAG 2.1 AA). Login surface (German default) → MFA TOTP challenge → JWT issued by studio-backend → role-aware nav rendered: `admin` sees everything; `developer` sees schema/Functions/deploys; `viewer` is read-only with all write affordances hidden. Role gates enforced server-side at the Gateway (F-CLI-3) — UI is the convenience layer.

## Acceptance criteria

- [ ] Login + MFA TOTP flow works for all three Roles
- [ ] Viewer Operator: UI hides Apply / Save / Deploy buttons; API returns 403 on write attempts (§6 #19)
- [ ] Developer Operator: can deploy Functions, apply migrations, override Pipeline (§6 #18)
- [ ] Admin Operator: full surface incl. Operator-management + Function-secrets + Vault read
- [ ] Light + dark mode toggle; SF-Pro-Display loaded, Inter fallback
- [ ] WCAG 2.1 AA conformance verified (axe-core CI gate, no critical violations)
- [ ] DE is default locale (browser-header / Operator preference per `_ntts.studio_users.locale`)

## Blocked by

- [02-setup-token-first-admin.md](./02-setup-token-first-admin.md)
