Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Catalog schemas (§4.8a). Shared meta-schema (`packages/catalog-schema/`, Zod) drives three catalog files in the image: `wrappers.catalog.json`, `oauth-providers.catalog.json`, `sms-providers.catalog.json`. Each file carries `schema_version: int`. Validation at three points: CI-Build, studio-backend boot (refuse-to-start on mismatch), Operator-Reload via `NOTIFY ntts_catalog_reload`. Operator extensions via `/etc/ntts/catalogs.d/*.json` (same schemas, merged at boot). Schema versioning tied to NTTS release: boot rejects unknown versions with sharp "NTTS upgrade required" hint.

## Acceptance criteria

- [ ] `packages/catalog-schema/` exports Zod schemas for all three catalog types
- [ ] CI Build validates all three image catalogs against their Zod schemas
- [ ] studio-backend refuses to start with unknown `schema_version`
- [ ] `/etc/ntts/catalogs.d/*.json` files merged on boot; collisions logged
- [ ] `NOTIFY ntts_catalog_reload` triggers in-process reload + re-validation
- [ ] Schema version bump in NTTS release requires Operator upgrade (clear error message)

## Blocked by

- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
- [33a-sms-catalog-twilio.md](./33a-sms-catalog-twilio.md)
- [44-wrappers-fdw-egress-proxy.md](./44-wrappers-fdw-egress-proxy.md)
