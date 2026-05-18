Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

i18n: DE (default) + EN (§4.11). Studio Frontend uses i18next with ICU MessageFormat. JSON catalogs in `apps/studio-frontend/src/i18n/{de,en}/*.json`. Default locale via browser-header or Operator preference on `_ntts.studio_users.locale`. EN strings are source-of-truth in code; DE professionally translated; catalog-sync via `i18next-parser` + translator workflow (Lokalise or Crowdin; OOR via PR). Auth-Email-Templates (slice 38) are DB-stored per-locale. **CLI + Logs + Audit-Strings + Error-Messages are EN-only** (CI-scripts grep output; CLI must be locale-stable). ICU MessageFormat handles plurals + dates (`{date, date, long}` → "18. Mai 2026" vs "May 18, 2026") + numbers.

## Acceptance criteria

- [ ] Studio renders DE strings when Operator locale is `de` (default); EN otherwise
- [ ] ICU date format renders "18. Mai 2026" for DE, "May 18, 2026" for EN
- [ ] Plural forms render correctly for both locales
- [ ] `i18next-parser` extracts EN source strings from code; CI fails if DE catalog has missing keys
- [ ] CLI output stays EN regardless of Operator locale
- [ ] Operator locale preference persisted to `_ntts.studio_users.locale`
- [ ] FR/ES/IT etc. accept community catalog contribution via PR (no release-block)

## Blocked by

- [07a-studio-shell-login-mfa-nav.md](./07a-studio-shell-login-mfa-nav.md)
