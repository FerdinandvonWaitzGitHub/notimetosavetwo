Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

`ntts fn scan` per F-CLI-7: License check against project allowlist (default MIT/Apache/BSD/PostgreSQL/ISC; configurable in `ntts.config.toml`); Bundle-size (warn >10MB, error >50MB); static-secrets regex scan on common API-key patterns committed in source → fail with line numbers; SBOM export `--out sbom.json` in CycloneDX or SPDX. Runs locally over Function source pre-deploy + is wired as a Pipeline stage (security-scan).

## Acceptance criteria

- [ ] `ntts fn scan ./functions/hello` reports licenses + sizes + static-secret hits + SBOM
- [ ] License outside allowlist → non-zero exit + clear message
- [ ] Bundle >50MB → exit 1 with message; >10MB → warn, exit 0
- [ ] Static-secrets regex matches common patterns (Stripe `sk_live_…`, AWS `AKIA…`, GitHub `ghp_…`, JWT `eyJ…`); reports file + line
- [ ] `--out sbom.json --format cyclonedx` writes valid CycloneDX; `--format spdx` writes valid SPDX
- [ ] Pipeline security-scan stage uses the same engine

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
