Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

DB-event trigger pathway per F-FN-3: SQL trigger calls `pg_net.http_post(...)` against `http://edge-runtime:9000/_internal/invoke/:name` with HMAC `hmac_f2f` header (from slice 4). Edge Runtime endpoint accepts the call only with valid HMAC, sets `as: 'service'` identity per F-FN-18 / ADR-0006, invokes the Function. Studio "DB Triggers" panel + CLI command surface the wiring; logs end up in `_ntts.function_logs` with the originating SQL context (table, op, request_id).

## Acceptance criteria

- [ ] AFTER INSERT trigger on `public.orders` fires pg_net → Edge Runtime → Function executes with `X-NTTS-Role: service_role`
- [ ] HMAC `hmac_f2f` validates; invalid HMAC returns 401
- [ ] Endpoint `_internal/invoke/:name` is unreachable through the Gateway
- [ ] Function log row in `_ntts.function_logs` carries the originating request_id
- [ ] Studio "DB Triggers" panel lists configured trigger → Function bindings
- [ ] CLI command lists/creates/removes trigger bindings
- [ ] pg_net errors (timeout, 5xx from Function) surface in trigger row + alert

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
