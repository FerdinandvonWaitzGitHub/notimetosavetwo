Status: ready-for-human

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Non-table Realtime channels (`broadcast`, `presence`) governed by `_ntts.realtime_channel_policies (id, topic_pattern, action enum('subscribe','publish'), expression, description)`. Default-deny: no truthy matching ALLOW policy → DENY. Expression grammar formally defined via pgsql-ast-parser-based whitelist validator. **Allowed**: comparison ops, boolean, jsonb-path-access, type-casts, whitelisted built-ins (`lower upper length coalesce array_position jsonb_path_exists starts_with split_part`), variables `request.jwt.claims` (jsonb) and `topic` (text). **Blocked**: subqueries, FROM/JOIN, DML, DDL, function-defs, dollar-quotes, CTEs, `WITH RECURSIVE`, non-whitelisted functions (esp. `pg_sleep`, `current_setting`). Validation at two points: Studio/CLI save (AST-walker reject with explanatory error) + runtime (`_ntts.eval_realtime_policy(expression, jwt_claims, topic)` SECURITY DEFINER that builds the WHERE clause and executes). Studio editor exposes a test-tool ("Would this topic + JWT-claims be allowed?") with inline grammar errors as you type. HITL because the grammar surface is genuinely novel — needs sign-off on Allow/Block lists, the test-tool UX, and the SECURITY DEFINER eval shape before lock-in.

## Acceptance criteria

- [ ] Allowed expression (`request.jwt.claims->>'org_id' = split_part(topic, ':', 1)`) saves and evaluates true/false correctly
- [ ] Disallowed expression (subquery, JOIN, `pg_sleep`, `current_setting`) rejected at save with explanatory error
- [ ] Default-deny: no matching ALLOW policy → DENY at subscribe + publish
- [ ] Test-tool returns Allow/Deny for arbitrary `(topic, jwt_claims)` input
- [ ] Grammar errors inline as you type
- [ ] `_ntts.eval_realtime_policy(...)` SECURITY DEFINER returns boolean
- [ ] Audit row written on every policy add/remove/edit

## Blocked by

- [25-realtime-postgres-changes.md](./25-realtime-postgres-changes.md)
