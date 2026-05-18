Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

GoTrue fork-patch (F-AUTH-12): fires `NOTIFY ntts_jwt_changed, '<user_id>'` on token refresh and on logout. Realtime container listens on this channel and evicts Subscriber-Authorization-Snapshot-Cache entries for the affected user (ADR-0026). Wires up the eviction logic on the Realtime side and confirms cache invalidation under load.

## Acceptance criteria

- [ ] GoTrue token refresh fires `NOTIFY ntts_jwt_changed, '<user_id>'`
- [ ] Logout fires the same notify
- [ ] Realtime container evicts L2 cache entries for that user_id within 100ms
- [ ] Test: revoke a user's session → next Realtime row-change for that user is denied (RLS re-evaluated against new JWT)
- [ ] No cache-eviction storm under 100 logouts/min

## Blocked by

- [25-realtime-postgres-changes.md](./25-realtime-postgres-changes.md)
- [30-oauth-catalog-driver-google.md](./30-oauth-catalog-driver-google.md)
