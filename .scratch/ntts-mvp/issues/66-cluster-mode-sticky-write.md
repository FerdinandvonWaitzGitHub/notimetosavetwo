Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Sticky-write per ADR-0025: after a write by a given session/JWT, subsequent reads from the same session are routed to the primary for a configurable window (e.g. 5s) to give the user read-your-writes consistency on PostgREST + direct connections. Implementation rides on Supavisor's session-state tracking (slice 65). Surfaced metric: per-session sticky-write windows in flight.

## Acceptance criteria

- [ ] After INSERT in session X, subsequent SELECT in session X within the window lands on primary
- [ ] Configurable window length per Operator (default 5s)
- [ ] After window expires, reads in session X may again land on replica
- [ ] `ntts_sticky_write_active` Prometheus gauge exposes count of sessions in sticky-window
- [ ] Service-role connections always sticky to primary (no replica reads)
- [ ] Documented "anonymous client" semantics (browser tab without persistent connection) — sticky-write only meaningful on long-lived sessions

## Blocked by

- [65-cluster-mode-supavisor-rw-split.md](./65-cluster-mode-supavisor-rw-split.md)
