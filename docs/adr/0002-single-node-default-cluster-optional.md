# Single-node default, cluster features optional

NTTS ships as a single-node appliance by default — one Postgres, one Realtime, one Edge Runtime, one Storage. Read replicas and multi-node Realtime exist as *optional cluster mode* but are not the recommended path. Single-node 99.5% (§7 NFR) describes the appliance promise; operators who need more pull the cluster levers explicitly.

## Considered Options

- **Multi-node by default with HA.** Rejected: kills "docker compose up" promise, needs orchestrator skills, contradicts §3 "appliance, not PaaS".
- **Single-node only, drop replicas and multi-node Realtime.** Rejected: blocks scale-up for serious operators, breaks parity with Supabase's read-replicas feature.
- **Single-node default, cluster opt-in (chosen).** Box stays an appliance; the scaling story is real but off the golden path.

## Consequences

API contracts assume a single writer. When cluster mode is engaged, the SDK URL is unchanged but reads can be routed at gateway level; write path stays primary-only. The Realtime multi-node story requires shared Phoenix PubSub — operator concern, documented in runbook.
