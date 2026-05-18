# Bundle manifest: TS types primary, extracted-AST secondary

Each Function Bundle carries a manifest (`_ntts.function_bundles.manifest jsonb`) recording which schema objects it touches. The manifest is the input to bidirectional schema-compat-check (F-FN-12). Primary correctness comes from the typed `db` client — the Function cannot tsc-compile against a schema that lacks columns it uses. Secondary, AST-extracted manifest captures table/column read/write sets, RPCs, extensions, and the dynamic-SQL flag for the reverse check (migration → blocked if affected Functions exist).

## Considered Options

- **Static AST extraction only.** Rejected: misses dynamic strings, breaks under common abstractions, requires no human guidance — false-negatives are silent.
- **Runtime tracing during test stage only.** Rejected: only as complete as the test suite; encourages "happy path coverage" thinking.
- **Type-system only.** Rejected: catches the forward check (deploy against new schema) but the reverse check (migration sees all affected Functions) needs explicit reads/writes recorded.
- **Hybrid (chosen).** tsc is the primary guarantee; AST manifest fills the reverse-direction gap. Dynamic SQL is acknowledged as a soft-warn path with audit, not silently allowed.

## Consequences

- Dynamic SQL bypasses the hard check. The manifest carries `"dynamic_sql": true` and the Pipeline emits a soft-warn that must be acknowledged in writing (lands in `_ntts.function_audit`).
- An ESLint rule (`no-dynamic-table-name`) makes dynamic SQL a deliberate, visible choice — not an accidental hole.
- Function-to-Function calls extend the effective manifest transitively at Bundle time. Cycles are rejected.
- The manifest is durable data. Changing its shape later requires a `_ntts` migration that re-derives manifests for the last 20 retained Bundles per Function, or accepts that pre-migration Bundles cannot be reverse-checked.
