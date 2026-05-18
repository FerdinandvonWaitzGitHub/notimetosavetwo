# Type-Gen flow: dedicated worker, async to migrations, versioned bundles with soft-recompile

NTTS generates two TypeScript declaration artefacts (`db.d.ts` from the live Postgres schema, `functions.d.ts` from Function bundle manifests). Generation runs in a dedicated `ntts-typegen` worker container, is asynchronous to the migration COMMIT, and produces a monotonic version counter plus a content fingerprint. Each Bundle pins the version it was authored against; deploy-time compat-check soft-recompiles against the current schema and audits any drift. Editors pull-on-open and receive WebSocket invalidations rather than push diffs. The DB row in `_ntts.generated_types` is the server-side source of truth; the filesystem `./types/db.d.ts` is the client-side mirror for local dev and IDE support.

## Considered Options

- **Inline plv8/plpython generator (atomic with migration).** Rejected: TS-string rendering inside Postgres is unmaintainable; generator code versioning becomes painful.
- **Generator lives in `studio-backend`.** Rejected: couples Editor UX availability to typegen correctness; failure of typegen blocks user-facing studio work.
- **Sync migration → typegen coupling.** Rejected: makes a typegen bug a migration-blocker; renders large-schema migrations user-visible-slow.
- **Single permissive type set, schema-shape (chosen for RLS handling).** Static analysis of arbitrary RLS expressions is effectively undecidable; matching Supabase semantics keeps the migration path clean and avoids fragile compile-time guesses.
- **Hard-fail deploy on schema drift.** Rejected: forces rebuild on every unrelated migration, kills productivity. Soft-recompile chosen instead.

## Mechanism

- **Container.** Dedicated `ntts-typegen` worker. Also runs the deploy-time Bundle-Compat-Check (F-FN-12) — same generator code, same fingerprint logic.
- **Implementation.** Eigene TypeScript-Implementierung in `apps/ntts-typegen/`. Catalog-Inspection via [`pg-meta`](https://github.com/supabase/pg-meta) (Supabase, MIT, TS), eigener Emitter (`ts-morph`-basiert) für `db.d.ts` und `functions.d.ts`. Erlaubt Co-Emission von Zod-Runtime-Schemas neben den TS-Types. Voll im Monorepo, keine Go-Toolchain im typegen-Container.
- **Triggers.** (a) Migration COMMIT (F-MIG-5 post-migration chain). (b) `pg_event_trigger` Backstop on raw DDL outside the migration path, bumping `_ntts.out_of_band_ddl_count` for audit-visibility. (c) Function-Bundle deploy events for `functions.d.ts`.
- **Artefacts.** Two rows in `_ntts.generated_types`: `name='db'` (schema-derivable: tables, views, enums, RPCs, composite types, storage-bucket-name union, realtime channel types) and `name='functions'` (F2F call signatures aggregated from each Bundle's manifest).
- **Version pointer.** Monotonic counter incremented atomically by the worker on every successful write. Schema fingerprint (SHA over a pg_catalog snapshot) stored alongside for verification and diagnostics.
- **Bundle manifest fields.** `types_version`, `types_fingerprint`, `recompiled_from` (set on soft-recompile, otherwise null).
- **Deploy-time race.** Soft-recompile: if current `types_version` > Bundle's, worker re-runs `tsc` against the live schema. Green → deploy with updated `types_version` and a `function.recompiled` audit event. Red → reject with diff.
- **Editor delivery.** Pull `_ntts.generated_types` on tab open. Subscribe to WS channel `types:changed`; receive version-bump notifications only (no type payload). Refetch on Save or short debounce.
- **Async coupling.** Migration COMMIT does not wait for typegen. Worker writes atomically (single transaction). On boot, worker compares stored fingerprint with the live schema fingerprint and regenerates on mismatch — catchup-on-restart, NOTIFY is a wake-up signal, not the only trigger.
- **RLS semantics.** Types reflect schema shape, not RLS-filtered shape. Generated-types header carries a disclaimer comment. Runtime is the source of truth for RLS filtering.

## Consequences

- A new container `ntts-typegen` joins the compose stack (still single-node per ADR-0002).
- `_ntts.generated_types(name, content, version, fingerprint, generated_at)` is a durable schema object; changes to its shape require a `_ntts` migration.
- Stale-types window between migration COMMIT and worker write is bounded by worker throughput; visible in audit as the latency between `migration.applied` and `types.regenerated` events.
- If `ntts-typegen` is fully down, Function deploys block (compat-check unavailable) — acceptable Operator-escalation path. Migrations themselves still succeed.
- Raw-DDL Backstop guarantees types track schema even when an Operator bypasses migrations, at the cost of `out_of_band_ddl_count` growing as a divergence indicator.
- Skipping RLS-aware types is a deliberate trade — power-user requests to opt-in to RLS-shape inference are bounded by Postgres' inability to statically analyse arbitrary policy SQL.
