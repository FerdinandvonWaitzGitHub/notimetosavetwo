Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

`ntts-postgres:full` image variant (§4.1) adding PostGIS, pgroonga, plv8, plpython to the `:slim` base. Same image layout, same boot semantics; Operator chooses by image tag. `:slim` (MIT-pure) remains default. Clear documentation: `:full` includes copyleft-touching extensions (PostGIS uses GPL components in some tools) — call out the license posture.

## Acceptance criteria

- [ ] `ntts-postgres:full` image contains PostGIS, pgroonga, plv8, plpython
- [ ] `SELECT ST_AsText(geom)` works on `:full`; returns clear "extension not installed" on `:slim` (§6 #22)
- [ ] Image size delta documented (`:full` will be ~500MB-1GB larger)
- [ ] License labelling on `:full` clear in docs + compose comments
- [ ] CI builds both `:slim` + `:full` for amd64 + arm64
- [ ] `ntts upgrade` correctly differentiates between `:slim` and `:full` and won't downgrade an Operator on `:full` to `:slim` accidentally

## Blocked by

- [59-ci-cd-multi-arch-cosign.md](./59-ci-cd-multi-arch-cosign.md)
