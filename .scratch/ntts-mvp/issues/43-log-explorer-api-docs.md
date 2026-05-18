Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

**Log explorer**: Studio panel + CLI `ntts fn logs` over `_ntts.function_logs` with filters (function_id, status, time range, request_id, free-text search), tail-mode, partitioned-table aware. **Reports** generated from log aggregates (per-Function invocation count, p95 latency, error rate). **Auto-generated API docs**: Studio page rendering OpenAPI spec from PostgREST `/openapi` (REST), pg_graphql introspection (GraphQL), and `_ntts.function_deployments` + Bundle manifests (Functions); links to per-endpoint examples.

## Acceptance criteria

- [ ] Studio Log Explorer filters work over millions of rows partitioned by day
- [ ] `ntts fn logs --function hello --tail` streams new log lines as they arrive
- [ ] `ntts fn logs --request-id <uuid>` returns the complete trace across DB-trigger, F2F, response
- [ ] Reports panel shows invocation count + p95 + error rate per Function for last 24h / 7d / 30d
- [ ] API docs page renders REST + GraphQL + Functions endpoints
- [ ] Per-Function manifest data (reads/writes/rpcs/extensions) surfaces in docs

## Blocked by

- [06-type-gen-worker.md](./06-type-gen-worker.md)
- [11-first-deployed-function.md](./11-first-deployed-function.md)
