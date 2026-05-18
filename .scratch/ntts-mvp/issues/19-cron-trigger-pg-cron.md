Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Cron trigger pathway per F-FN-4: pg_cron schedule executes a SQL that calls pg_net → `http://edge-runtime:9000/_internal/invoke/:name` with `hmac_f2f`, identity `service_role`. Cron schedules live in `_ntts.function_cron_schedules (function_id, cron_expr, enabled, created_by, created_at)` and are mirrored to pg_cron jobs. Studio + CLI surface the schedule list. Logs land in `_ntts.function_logs` with `execution_id` traceable from the cron run.

## Acceptance criteria

- [ ] `*/5 * * * *` schedule on a Function fires within 1 min of the next interval boundary
- [ ] `_ntts.function_logs` row appears with the cron's `execution_id` (§6 #4)
- [ ] Disabling a schedule in Studio removes the corresponding pg_cron job
- [ ] CLI commands list / add / remove cron schedules
- [ ] Capacity-target: 100 active pg_cron jobs supported (§7)

## Blocked by

- [18-db-event-trigger-pg-net.md](./18-db-event-trigger-pg-net.md)
