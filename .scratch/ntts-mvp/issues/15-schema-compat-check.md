Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Bidirectional schema-compat-check per ADR-0005. **Primary** = `tsc` against the live schema's generated `db`-types; deploy-stage runs in `ntts-typegen` worker; compares Bundle's `types_version` against current. If drifted, soft-recompile against current schema. Green → deploy with `recompiled_from` audit field. Red → reject with diff. **Secondary** = Bundle manifest in `_ntts.function_bundles.manifest jsonb` with `{schema_hash, reads, writes, rpcs, extensions, function_to_function, dynamic_sql}`. **Forward (deploy)**: Bundle manifest vs current schema → block if needed columns missing. **Reverse (migration)**: planned DDL vs union of active Bundles' manifests → block + list affected Functions + offer 1-click "sync all" (re-runs Pipeline against new schema in parallel). Dynamic SQL → manifest carries `"dynamic_sql": true` → soft-warn, not hard-block; override goes to `_ntts.function_audit`. ESLint rule `no-dynamic-table-name`. Function-to-Function calls merge transitively at Bundle time; cycles → hard-fail.

## Acceptance criteria

- [ ] Bundle manifest persisted in `_ntts.function_bundles.manifest` on every deploy
- [ ] Migration dropping a column used by ≥1 deployed Function is blocked; affected Functions listed (§6 #9)
- [ ] 1-click "sync affected Functions" opens editor for listed Functions and queues re-deploy (F-MIG-10)
- [ ] Dynamic-SQL Function deploys with `dynamic_sql: true` and soft-warn; override lands audit
- [ ] ESLint rule `no-dynamic-table-name` raises in Function source
- [ ] Function-to-Function cycle → hard-fail at Bundle time
- [ ] Drifted `types_version` triggers soft-recompile; success deploys with `recompiled_from` audit field

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
- [05-migration-apply-and-schema-cache-reload.md](./05-migration-apply-and-schema-cache-reload.md)
