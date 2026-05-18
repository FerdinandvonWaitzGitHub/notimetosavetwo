Status: ready-for-human

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Cluster Mode: Supavisor as read/write splitter per ADR-0025 (Supavisor fork — Elixir). Statement inspection decides reads vs writes; reads go to replicas, writes to primary. PostgREST and direct clients all go through the pooler; HTTP layer stays oblivious. Operator opts in via `NTTS_CLUSTER_MODE=true` + replica config. HITL because (a) statement-inspection rule set needs sign-off (which PG-statements count as "write"?), (b) replica-lag tolerance needs Operator-policy design, (c) PG hot-standby setup is sharp-edged operationally.

## Acceptance criteria

- [ ] Supavisor fork classifies a sample workload (SELECT vs INSERT/UPDATE/DELETE/DDL/CTE w/ writes) >99% correctly
- [ ] Read replicas can be added/removed via Operator config without Supavisor restart
- [ ] PostgREST traffic routed through Supavisor; reads land on replica when fresh enough
- [ ] Operator picks replica-lag tolerance (e.g. max 5s); stale replica skipped
- [ ] Failover: primary failure → Supavisor routes all traffic to replicas until Operator promotes

## Blocked by

- [24-supavisor-pooler.md](./24-supavisor-pooler.md)
