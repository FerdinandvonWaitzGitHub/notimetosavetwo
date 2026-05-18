Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Sandbox quotas per F-FN-5 + F-FN-9: timeout default 30s, max 150s; CPU 5s soft / 30s hard; mem 256MB / 1GB; disk 50MB tmpfs; egress allowlist; process-spawn block. Quota-kill = V8 terminates the offending user-worker; parallel invocations on their own isolates untouched. Background tasks via `waitUntil` (Edge Runtime native) extend quotas by 30s post-response. Stdout/stderr go to `_ntts.function_logs` (range-partitioned daily by `started_at`) with `execution_id, function_id, version_id, started_at, duration_ms, cpu_ms, memory_peak_kb, exit_status`.

## Acceptance criteria

- [ ] `while(true){}` Function → user-worker killed in ≤30s (§6 #7)
- [ ] Parallel invocations during a kill are unaffected (§6 #7)
- [ ] Mem 1GB hard limit triggers OOM kill with structured error
- [ ] Egress to disallowed host returns network error; allowed host succeeds
- [ ] `child_process.spawn` blocked from user code
- [ ] `waitUntil(promise)` extends quota by 30s post-response
- [ ] `_ntts.function_logs` carries all the F-FN-6 columns; partitioned daily

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
