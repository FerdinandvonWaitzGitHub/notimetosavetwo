Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Optional continuous mirror of audit tables (F-LOG-2). `NTTS_AUDIT_DRAIN=s3://...` or `syslog://...` configures the sink. Each sink-write lands `_ntts.audit_drain_receipts (table_name, partition, sink_id, written_at, row_count, ...)` so we can prove what was drained and when. Drain workers run as a studio-backend goroutine/timer; back-pressure causes WARN alert (F-LOG-4) at 60min lag, ERROR at 1440min.

## Acceptance criteria

- [ ] `NTTS_AUDIT_DRAIN=s3://bucket/path` writes audit rows to S3 in near-real-time
- [ ] `NTTS_AUDIT_DRAIN=syslog://host:port` writes audit rows to syslog
- [ ] Every successful drain writes an `_ntts.audit_drain_receipts` row
- [ ] Drain lag >60min → WARN alert; >1440min → ERROR alert
- [ ] Drain failures retried with backoff; permanent failures alert without traffic block
- [ ] Drain receipts cover all three audit tables (`studio_audit_read/write`, `function_audit`)

## Blocked by

- [10-audit-chain-integrity.md](./10-audit-chain-integrity.md)
