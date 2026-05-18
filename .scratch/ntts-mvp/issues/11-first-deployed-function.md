Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Edge Runtime forked (Rust) and wrapped: V8 user-worker isolates, one container, fixed concurrency. Bundle is the eszip-compiled immutable artifact stored in `_ntts.function_bundles (id, function_id, hash, eszip_bytes, manifest jsonb, signature_pipeline, signature_operator nullable, created_at, created_by)`. Pointer in `_ntts.function_deployments.active_version_id`. 8-stage Pre-deploy Pipeline (tsc → eslint → bundle → test → schema-compat → security-scan → signature → dry-run-deploy) runs end-to-end for every deploy (override flow is slice 12). Signature stage uses Ed25519 `NTTS_BUNDLE_SIGNING_KEY` over `(bundle_hash ‖ manifest_hash ‖ pipeline_run_id ‖ unix_timestamp)`. Atomic deploy: one transaction INSERT Bundle + UPDATE pointer + `NOTIFY ntts_fn_deploy, '{function_id}'`. Main worker holds `LISTEN ntts_fn_deploy` → fetch eszip → write tmpfs → evict isolates → next invocation recompiles. Boot hydration: read all active deployments, load Bundles to tmpfs, then start LISTEN; single broken Function → `status: 'broken'`, others stay up. `POST /functions/v1/:name` invokes a public Function. `pg_dump` covers `function_bundles`, `function_deployments`, `function_versions`, `function_audit`.

## Acceptance criteria

- [ ] `ntts fn deploy hello` runs all 8 Pipeline stages green; Bundle lands in `_ntts.function_bundles`; pointer set
- [ ] `POST /functions/v1/hello` returns the Function's response (§6 #11)
- [ ] Deploy → cache invalidation propagates in <500ms via NOTIFY + bundle fetch (§6 #11)
- [ ] Signature-pipeline column populated; pointer-UPDATE refused if signature invalid or NULL
- [ ] Boot hydration loads all active deployments to tmpfs before LISTEN starts
- [ ] Single broken Function marks `status: 'broken'`; others stay up
- [ ] `pg_dump` round-trip restores all Functions invokable (§6 #12 partial)

## Blocked by

- [05-migration-apply-and-schema-cache-reload.md](./05-migration-apply-and-schema-cache-reload.md)
- [04-internal-hmac-bootstrap.md](./04-internal-hmac-bootstrap.md)
