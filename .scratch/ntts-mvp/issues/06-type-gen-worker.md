Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

New `ntts-typegen` container holding `LISTEN ntts_typegen`. On every migration COMMIT, studio-backend fires `NOTIFY ntts_typegen`; the worker regenerates `_ntts.generated_types` rows (`name='db'` and `name='functions'`), bumps `types_version`, writes new `types_fingerprint`. `pg_event_trigger` Backstop catches raw DDL outside the migration path (out-of-band path; increments `_ntts.out_of_band_ddl_count`). `ntts-typegen` publishes WS event `types:changed` so Studio Editor pulls fresh types on next save/debounce. CLI `ntts types generate` and `ntts types watch` mirror the same fetch.

## Acceptance criteria

- [ ] `ntts-typegen` container in compose; depends on Postgres healthy
- [ ] Migration COMMIT fires `NOTIFY ntts_typegen`; worker regenerates within 5s for ≤100-table schema
- [ ] `_ntts.generated_types` has rows `name='db'` and `name='functions'` with `types_version` and `types_fingerprint`
- [ ] `pg_event_trigger` Backstop catches out-of-band DDL and triggers regen; `_ntts.out_of_band_ddl_count` increments
- [ ] WS event `types:changed` reaches Studio Editor; types update on next save/debounce
- [ ] `ntts types generate` writes types file per `ntts.config.toml`; `ntts types watch` re-emits on changes
- [ ] Long type-gen runs do not block studio-backend API requests (decoupled per ADR-0012)

## Blocked by

- [05-migration-apply-and-schema-cache-reload.md](./05-migration-apply-and-schema-cache-reload.md)
