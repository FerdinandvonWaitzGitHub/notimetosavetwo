# Realtime: default-deny, RLS-derived for table channels, policy-engine for broadcast/presence

NTTS Realtime takes a default-deny stance to close the well-known Supabase Realtime/RLS coordination gap. Table-change broadcasts (`postgres_changes`) require explicit per-table opt-in via `_ntts.realtime_enabled_tables`, and enabled tables broadcast only events the subscriber would have been allowed to SELECT under RLS. Non-table channels (`broadcast`, `presence`) require explicit Operator-authored policies in `_ntts.realtime_channel_policies`; no policy match → no subscribe, no publish. RLS remains the single source of truth for row visibility; channel policies are the analogous truth for non-table topics.

## Considered Options

- **Default-allow without RLS filter (Supabase legacy).** Rejected: privacy hazard, non-starter under EU sovereignty positioning.
- **Default-allow with RLS filter.** Rejected: an Operator who forgets RLS on a table silently starts broadcasting everything to everyone. Default-deny matches the security stance taken elsewhere.
- **Separate channel policies authored alongside RLS for table channels.** Rejected: duplicates the source of truth (table RLS) and creates drift surface. RLS-derived is the only way to keep them aligned.
- **Default-allow broadcast/presence for authenticated users.** Rejected: too coarse; room-isolation (User cannot publish to other Users' rooms) requires topic-pattern awareness.

## Mechanism

### Table channels (`postgres_changes`)
- **Opt-in table.** `_ntts.realtime_enabled_tables (schema, table, enabled_at, enabled_by, requires_rls_check bool default true)`. Pre-toggle UI check verifies RLS is enabled on the table and at least one policy exists; otherwise toggle stays off with helper "Enable RLS first".
- **Realtime service** loads enabled set on boot, `LISTEN ntts_realtime_table_change` for live updates.
- **Broadcast filter.** Per row-change event, per subscriber: `SET LOCAL role …; SET LOCAL request.jwt.claims …; SELECT * FROM <table> WHERE <pk> = <changed_pk> LIMIT 1;` Zero rows → skip broadcast. Non-zero → broadcast returned columns (RLS column-level filtering honoured via views).
- **Service-role subscribers.** Connections with service-JWT (Functions, internal) bypass RLS as normal Postgres semantics dictate — they see all events.
- **Performance.** Naive shape is N×M (N subscribers × M events). Optimisations (group subscribers by role+claims, cache policy evaluation per (row, role-bucket)) come from Supabase's Realtime codebase but must be CI-tested for correctness under JWT-claim variance.

### Non-table channels (`broadcast`, `presence`)
- **Policy table.** `_ntts.realtime_channel_policies (id uuid, topic_pattern text, action enum('subscribe','publish'), expression text, description text, created_by, created_at, updated_at)`.
- **Default-deny.** No matching ALLOW policy → DENY. Explicit DENY policies are a future addition; v1.0 has only ALLOW semantics.
- **Expression language.** PostgreSQL boolean expression subset, evaluated read-only, with access to `request.jwt.claims` and a special `topic` variable. No DML, no FROM/JOIN — only boolean over JWT-claims + topic. Validator rejects expressions that contain non-allowed constructs.
- **Evaluation.** Per connection-open and per topic-join, Realtime service evaluates applicable policies (matched by `topic_pattern` glob/regex and `action`). At least one truthy ALLOW required to proceed.
- **Denies audit.** Sampled deny events written to `_ntts.realtime_policy_denies (topic, action, jwt_sub, denied_at, sample_rate)`. Rate-limited to N rows/sec to prevent log-flooding under malicious traffic. Partitioned + retention via F-LOG-1 retention pattern, not part of audit-immutable hash chain.
- **Studio UI.** Policies editor with PostgreSQL syntax highlighting (analogous to RLS-policy editor); "Would this topic + JWT-claims be allowed?" test tool.

### Auth integration
- Realtime container trusts `X-NTTS-Auth` header set by `ntts-edge` (ADR-0004 single-gateway-JWT-validated-once). Realtime's own JWT-verification is disabled. The header is signed by `ntts-edge` with an internal HMAC key, verified by Realtime, populated into the `request.jwt.claims` SET LOCAL for policy/RLS evaluation.

## Consequences

- A behavioural difference from Supabase Cloud: NTTS users migrating from Supabase must opt in tables to Realtime explicitly. Documented as a migration step.
- The `_ntts.realtime_enabled_tables` and `_ntts.realtime_channel_policies` tables become part of `pg_dump` backup scope; Realtime config is restorable.
- Realtime container's local JWT-verification configuration becomes a static internal-HMAC trust relationship with ntts-edge. Rotation of that internal HMAC is part of NTTS upgrade flow.
- Channel-policy expression engine introduces a small DSL that must be validated and CI-tested against injection and side-effect attempts. The expression validator is part of the studio-backend.
- Default-deny biases the Operator toward thinking about each Realtime surface intentionally — desirable for sovereignty positioning, mildly painful for fast prototyping. CLI scaffolding (`ntts realtime enable <table>`, `ntts realtime policy add ...`) keeps the friction low.
- Performance is bounded by RLS evaluation cost per event×subscriber. Operators with high-volume Realtime workloads must size Postgres accordingly; NTTS does not promise unbounded fanout on the single-node appliance.
