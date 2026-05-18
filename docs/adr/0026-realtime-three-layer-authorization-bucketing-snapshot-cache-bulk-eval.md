# ADR-0026: Realtime Three-Layer Authorization — Topic-Filter Bucketing + Subscriber-Snapshot-Cache + Bulk-Eval-Fallback

**Status:** Accepted
**Date:** 2026-05-18
**Resolves:** F-RT-2 Performance-Gap (Capacity-Targets §7: 5.000 Realtime Subscribers auf 4-vCPU/8GB)

## Context

F-RT-2 spezifiziert die Realtime-Authorization-Semantik sauber: pro Row-Change × pro Subscriber, Realtime setzt JWT-Context und führt PK-`SELECT` gegen die geänderte Row aus (Zero Rows → skip broadcast, sonst broadcast der RLS-gefilterten Spalten).

Diese Semantik ist korrekt aber **skaliert nicht zum Capacity-Target**: 50 Row-Changes/s × 1.000 Subscriber auf einer hot table = 50.000 RLS-`SELECT`s/s. Postgres auf 4 vCPU schafft realistisch 10–30k einfache PK-`SELECT`s/s. Bei 5.000 Subscriber × 50 Changes/s wären es 250.000 `SELECT`s/s — Postgres-Tod.

Das Capacity-Target §7 (5.000 Realtime Subscriber single-node) ist ohne architektonische Optimierung nicht haltbar. F-RT-2 als reine Per-Subscriber-RLS-Eval-Schleife ist die saubere Default-Semantik, aber sie braucht eine Performance-Schicht darüber.

## Considered Options

1. **Per-Subscriber-Cache (Result-Memoization).** Cache pro Subscriber × Row-PK den letzten Authorization-Boolean für N Minuten. Invalidation auf JWT-Refresh oder Row-Update.
   - **Verworfen als Single-Solution:** Hot-Tables mit häufigen Row-Updates invalidieren ständig; löst nicht den per-Change-Fan-out, sondern nur den Steady-State.

2. **Bulk-RLS-Eval als Postgres-side-Loop.** Pro Row-Change eine einzige Query, die intern die Subscriber-Liste iteriert und je Subscriber die RLS-Policy gegen das Row-JSONB evaluiert.
   - **Verworfen als Single-Solution:** funktioniert für komplexe Policies, aber für die 90% simpler Equality-Policies (`user_id = auth.uid()`) ist es Overkill — Postgres-Round-Trip wo In-Memory-Vergleich genügt hätte.

3. **Topic-Filter-Bucketing.** Subscriber gruppieren nach `(channel, filter_hash)`, Broadcast pro Bucket statt pro Subscriber.
   - **Verworfen als Single-Solution:** funktioniert nur für identische Filter (klassisch: alle in einem Group-Chat). Per-User-RLS hat per Definition unique Filter pro Subscriber — Bucketing greift nicht.

4. **Drei-Schichten-Synthese (gewählt).** Jede der drei Patterns adressiert einen anderen Workload-Charakteristik; kombiniert decken sie >95% realer RLS-Patterns mit minimaler Postgres-Last.

5. **Single-Node-Target reduzieren (Workaround statt Architektur).** Alternative wäre, §7 von 5k auf 1k Subscriber zu senken und die Workload-Frage zu vertagen.
   - **Verworfen:** Capacity-Targets stehen per Entscheidung 2026-05-18, müssen architektonisch getragen werden.

## Decision

**Drei-Schichten-Authorization-Pipeline im Realtime-Container:**

### Layer 0 — Row-Change-Event (unverändert)

Logical-Replication-Slot liefert OLD/NEW JSONB an die Realtime-Node. Standard supabase-realtime Pfad.

### Layer 1 — Topic-Filter Bucketing

- Subscriber-Subscription wird intern als `(table_oid, filter_hash) → subscriber_set` geführt. `filter_hash` ist der stable Hash der client-supplied filter-Expression (PostgREST-Filter-DSL-Syntax aus dem `subscribe`-Aufruf von `supabase-js`).
- ETS-Tabelle `realtime_buckets`: `{table_oid, filter_hash} → [subscriber_id]`.
- Bei Row-Change: in-memory ermittle, welche `filter_hash`-Buckets mit dem Row matchen (Filter wird als Closure evaluiert auf das NEW-JSONB, Microseconds).
- Ein gematchter Bucket → eine weitergehende Authorization-Eval-Operation, **nicht eine pro Subscriber im Bucket**.
- **Hebel:** 1.000 Group-Chat-Subscriber mit identischem Filter `channel_id=eq.X` → 1 Authorization-Operation pro Row-Change in dem Channel.

### Layer 2 — Subscriber-Authorization-Snapshot-Cache

- Beim Subscribe (oder bei JWT-Refresh) holt Realtime aus Postgres genau einmal pro Subscriber eine **Snapshot-Expression**: eine Filter-Expression, die beschreibt was dieser Subscriber mit seinen aktuellen Claims auf dieser Tabelle sehen darf.
- Beispiel: RLS-Policy `user_id = auth.uid()` + Subscriber-Claims `sub='alice'` → Snapshot `user_id = 'alice'`.
- Snapshot-Storage: ETS-Tabelle `realtime_authz_snapshots` keyed durch `(jwt_claims_hash, table_oid, policy_version)`.
- Snapshot wird in-process vom Postgres-Side via SECURITY DEFINER-Function `_ntts.derive_authz_snapshot(jwt_claims jsonb, table_oid oid)` erzeugt, die intern die Policy-Definition introspect und die Filter-Expression mit substituierten Claims als jsonb-encoded Snapshot zurückgibt.
- Invalidation:
  - JWT-Refresh: GoTrue fires `NOTIFY ntts_jwt_changed, '<user_id>'` → Realtime evicted alle Snapshots dieses Subscribers.
  - Policy-Change: F-MIG-5 erweitert um `NOTIFY ntts_policy_changed, '<table_oid>'` → Realtime evicted alle Snapshots dieser Tabelle.
- Bei Row-Change innerhalb eines Buckets (Layer 1): pro Subscriber im Bucket wird die Snapshot-Expression **in-memory** gegen das NEW-JSONB evaluiert. Simple Equality-Vergleich, Microseconds pro Subscriber.
- **Hebel:** 90% realer Postgres-RLS-Policies sind Equality auf Claim-Fields (`user_id`, `tenant_id`, `org_id`). Für die fällt jede Authorization-Eval auf In-Memory-Vergleich zurück, Zero SQL-Queries.

### Layer 3 — Bulk-RLS-Eval als Fallback

- Snapshot-Cache-Miss bei Layer 2: die Policy ist nicht als statische Snapshot-Expression darstellbar (Subquery in Policy-Body, zeitabhängig via `now()`, abhängig von `current_setting()`, abhängig von Cross-Table-Lookup, etc.).
- Realtime sendet **eine** Query gegen Postgres:
  ```sql
  SELECT subscriber_id
  FROM _ntts.eval_row_visibility($1::jsonb[], $2::jsonb, $3::oid)
  ```
  - `$1` = Array von Subscriber-Context-JSONBs (jwt_claims pro Subscriber, vom Bucket).
  - `$2` = NEW-Row als JSONB.
  - `$3` = table_oid.
- `_ntts.eval_row_visibility` ist SECURITY DEFINER. Iteriert intern über das Subscriber-Array, pro Iteration: `SET LOCAL request.jwt.claims = $i`, dann Policy-Check via `SELECT TRUE WHERE EXISTS (SELECT 1 FROM <table> WHERE pk_columns = NEW.pk_columns)`. Accumulator akkumuliert berechtigte subscriber_ids in TEMP TABLE oder RETURNS TABLE.
- **Hebel:** 1 Round-Trip statt N. Postgres-Function-Call-Iteration ist um Größenordnungen billiger als N separate Connection-Acquisitions + Statement-Parses.

### Bucket-Match-Reihenfolge

Pro Row-Change-Event:

1. Layer 1 ermittelt matchende Buckets in-memory (Microseconds).
2. Pro gematchtem Bucket: Subscriber-Liste iterieren.
3. Pro Subscriber: Layer-2-Snapshot-Lookup. Hit → In-Memory-Eval gegen NEW-Row, Result als boolean.
4. Subscriber mit Layer-2-Miss → sammeln in Layer-3-Fallback-Batch.
5. Layer-3-Batch: eine SQL-Query, Subscriber-IDs zurück.
6. Final: Broadcast NEW-Row-Payload an alle berechtigten Subscriber.

## Consequences

### Lastmodell

| Subscriber-Pattern | Layer | Postgres-SQL-Queries pro Row-Change |
|---|---|---|
| Alle mit identischem Filter (Group-Chat, public board) | L1 → L2 cached | 0 (in-memory bucketed broadcast) |
| Per-User-RLS (`user_id = auth.uid()`) | L2 cached | 0 (in-memory per-subscriber eval gegen Snapshot) |
| Per-Tenant-RLS (`tenant_id = X`) | L2 cached | 0 (Snapshot ist Equality auf Claim-Field) |
| Komplexe Policy (Subquery, zeitabhängig, current_setting) | L3 fallback | 1 (Bulk-Eval pro Row-Change) |
| JWT gerade gerefreshed, Snapshot evicted | L2 re-fetch | 1 pro Subscriber bis Snapshot wieder cached |

Mixed-Workload (Annahme 70% L2-cached, 25% L1-bucketed-cached, 5% L3-fallback) bei 50 Changes/s × 1.000 Subscriber: ~10–50 SQL-Queries/s statt 50.000. **Drei Größenordnungen Reduktion.**

### Realtime-Fork-Patches (Elixir)

- **Bucket-Index-Modul** (~200 LOC): ETS-Tabelle, Subscribe/Unsubscribe-Hooks, Filter-Match-Closure.
- **Snapshot-Cache-Modul** (~500 LOC): ETS-Storage, Postgres-Fetch-Path, NOTIFY-Listener für Invalidation, Eviction-Policy (LRU bei Memory-Druck).
- **Bulk-Eval-Path** (~200 LOC): Batch-Collector, Postgres-Query-Sender, Result-Demuxer.

### `_ntts`-Schema-Erweiterungen

- `_ntts.derive_authz_snapshot(jwt_claims jsonb, table_oid oid) returns jsonb` (SECURITY DEFINER). Liefert eine Filter-Expression als JSONB-encoded Expression-Tree, der vom Realtime-Container in-memory evaluiert werden kann. Returns NULL wenn Policy nicht als statischer Snapshot darstellbar (zwingt L3-Fallback).
- `_ntts.eval_row_visibility(subscriber_contexts jsonb[], row_data jsonb, table_oid oid) returns TABLE(subscriber_id uuid)` (SECURITY DEFINER). Bulk-Eval-Function.
- Beide Functions in der kanonischen `_ntts.*` Init-Migration (F-UPG-1 `0001_init.sql`).

### NOTIFY-Channel-Erweiterungen

- `ntts_jwt_changed` (Payload: `user_id`) — von GoTrue gefeuert bei Token-Refresh und Logout. Realtime-Listener evicted Subscriber-Snapshots.
- `ntts_policy_changed` (Payload: `table_oid`) — von F-MIG-5 erweiterte post-migration Chain gefeuert. Realtime-Listener evicted alle Snapshots auf der Tabelle.

### Memory-Budget

L2-Snapshot-Cache: ~500 Bytes pro `(subscriber, table)` (JSONB-encoded Expression-Tree). 5.000 Subscriber × 10 abonnierte Tabellen = 50.000 Snapshots ≈ **25–50 MB ETS**. Innerhalb des Realtime-Container-Budgets aus §7-Kapazitäts-Rechnung.

L1-Bucket-Index: `(table_oid, filter_hash) → subscriber_id_list`. Bei 5.000 Subscriber und vielen unique Filtern: O(subscriber_count) Memory, ~1 MB ETS.

### Acceptance-Test-Ergänzung

§6 erweitert um:

> **#26:** Mit 1.000 Subscribers auf einer Tabelle mit RLS `user_id = auth.uid()` und 50 INSERTs/s durch Users X1..X50: jeder Subscriber sieht nur eigene Rows; Postgres-Load <500 RLS-Queries/s (gemessen via `pg_stat_statements`).

### Was die Architektur nicht löst

- **Truly dynamische Policies** (zeitabhängig via `now()`, mit Cross-Table-Subqueries, mit `current_setting()`-State) fallen permanent auf Layer 3. Wenn die Mehrheit der Operator-Policies dieser Form ist, kollabiert die Skalierung zurück auf F-RT-2-Baseline. **Mitigation:** Advisor (§4.6) markiert solche Policies als `realtime_uncached` Performance-Warnung mit Erklärung, was alternative Policy-Formulierung wäre.
- **Sehr individuelle Filter ohne RLS-Aggregation** (1.000 Subscriber jeder mit unique client-supplied Filter und unique RLS-Position) bleibt L1-uneffektiv. Real seltener Worst-Case.
- **JWT-Massenrefresh-Storm** (z.B. nach Auth-Service-Restart): viele gleichzeitige Snapshot-Re-Fetches. **Mitigation:** Coalesce per-user re-fetch via single-flight pattern im Realtime-Container; Re-Fetch-Queue mit Rate-Limit.

### Operator-UX

Studio "Realtime"-Panel (F-RT-7) zeigt pro abonnierter Tabelle:

- **Cache hit rate** (L2-cached vs L3-fallback) der letzten 5 Minuten.
- **Active subscriber count** pro `(channel, filter_hash)`-Bucket.
- **Slowest-evaluating policies** (Top 10 nach mittlerer L3-Eval-Zeit).

Diese Telemetrie schließt die Beobachtbarkeits-Lücke, die F-RT-5 (denies sampling) heute nur für Deny-Cases adressiert.

## Related

- F-RT-2 (RLS-derived broadcast — diese ADR ergänzt die Performance-Schicht, ändert nicht die Sicherheits-Semantik).
- F-RT-7 (Studio Realtime UI — wird um Cache-Telemetrie erweitert).
- F-MIG-5 (post-migration NOTIFY-Chain — wird um `ntts_policy_changed` erweitert).
- F-AUTH-1..3 (GoTrue JWT-Lifecycle — wird um `ntts_jwt_changed` NOTIFY auf Refresh+Logout erweitert).
- §7 Capacity-Targets (5.000 Realtime Subscriber — diese ADR macht das Target architektonisch tragbar).
- Future ADR: Realtime Multi-Node Distribution (Cluster Mode — Bucket-Index + Snapshot-Cache muss zwischen Nodes synchronisiert werden, eigene Komplexitäts-Schicht).
