Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Versioning + 1-click rollback: keep last 20 Bundles per Function. Rollback is a pointer-UPDATE in `_ntts.function_deployments.active_version_id` to a previous Bundle + `NOTIFY ntts_fn_deploy` + audit row with reason. `ntts fn rollback <name> [--to <bundle_id>]` from CLI; Studio Function page exposes a rollback action with reason prompt. Same atomic transaction as deploy.

## Acceptance criteria

- [ ] Rollback v17→v16: all invocations on v16 within 30s (§6 #10)
- [ ] Audit row written for rollback with reason (§6 #10)
- [ ] Last 20 Bundles per Function retained; older Bundles pruned
- [ ] CLI `ntts fn rollback <name> --to <bundle_id>` works; without `--to` rolls back to immediate previous
- [ ] Studio rollback action requires reason prompt (free-text)
- [ ] Cannot rollback to a Bundle with invalid `signature_pipeline`

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
