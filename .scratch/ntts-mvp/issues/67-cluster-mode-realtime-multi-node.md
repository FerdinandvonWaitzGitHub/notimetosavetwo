Status: ready-for-human

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Cluster Mode: Realtime multi-node distribution (Phoenix). supabase-realtime fork already in slice 25 — this slice wires the multi-node deployment: clustered BEAM nodes, channel-broadcast across nodes, presence sync via PG (or Phoenix.PubSub cluster), HMAC trust extended to all Realtime nodes. Operator opts in via `NTTS_CLUSTER_MODE=true` + replica + Realtime-node config. HITL because: (a) Phoenix clustering model + libcluster strategy needs explicit Operator-facing design (DNS vs Postgres-discovery), (b) presence + broadcast guarantees under partition need spelled-out Operator promise, (c) HMAC fan-out + rotation against N nodes is sharp-edged.

## Acceptance criteria

- [ ] 3-node Realtime cluster successfully shares presence + broadcast state
- [ ] Subscriber connected to node A receives broadcast initiated on node B
- [ ] HMAC rotation propagates to all nodes in <2s
- [ ] Single-node failure: subscribers reconnect to remaining nodes; presence reconciles
- [ ] Documented split-brain behaviour + Operator's guidance for healing
- [ ] §7 5.000-subscriber target achieved across cluster (not just single node)

## Blocked by

- [25-realtime-postgres-changes.md](./25-realtime-postgres-changes.md)
- [65-cluster-mode-supavisor-rw-split.md](./65-cluster-mode-supavisor-rw-split.md)
