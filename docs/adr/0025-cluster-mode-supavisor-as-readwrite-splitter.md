# ADR-0025: Cluster Mode — Supavisor als Read/Write-Splitter

**Status:** Accepted
**Date:** 2026-05-18
**Resolves:** PRD §10, Open Question 6

## Context

Cluster Mode (Read-Replicas + Multi-Node-Realtime) ist explizit in [ADR-0002](./0002-single-node-default-cluster-optional.md) als optional + opt-in beschrieben. Wenn Operator es aktiviert, muss klar sein, *wo* die Read/Write-Splitt-Logik lebt. Drei Kandidaten:

1. **Gateway-Level** (`ntts-edge`): HTTP-Method-Sniffing, GET → Replica, POST/PATCH/DELETE → Primary.
2. **PostgREST-Doppel-Instanz**: zwei PostgREST-Container, einer gegen Primary, einer gegen Replica. Gateway routet anhand Header.
3. **Supavisor**: der Pooler kennt das Postgres-Protokoll und kann pro-Statement (oder pro-Transaction) routen.

## Decision

**Supavisor ist der Read/Write-Splitter im Cluster Mode.**

Mechanik:
- Supavisor-Konfiguration deklariert Primary + N Replicas mit Latency-Health-Probes.
- Transaction-Mode-Connections mit `read_only=true` Session-Setting → Read-Replica-Pool.
- Default-Mode: `BEGIN` Probing — Supavisor inspiziert das erste Statement; wenn nur `SELECT/EXPLAIN/SHOW`, route auf Replica; bei `INSERT/UPDATE/DELETE/CREATE/ALTER/CALL/DO` → Primary.
- PostgREST verbindet ausschließlich über Supavisor (heute schon Default-Path-Empfehlung); GET-Requests führen zu Read-only-Connection-Akquise, POST/PATCH/DELETE zu Read-Write-Akquise.
- pg_graphql identisch, weil es als Postgres-Extension durch dieselbe PostgREST-Connection geht.
- Direct-DB-Clients (Operator-Skripte, externe Apps) zeigen weiterhin auf Supavisor `:6543` — Splitting transparent.
- App-User-Apps via `@supabase/supabase-js` ändern *keinen* Code; sie reden nur Gateway → PostgREST → Supavisor.

## Considered Options

1. **Gateway-Level-Routing.** Verworfen: kann RPC-POST (read-shaped) nicht von INSERT-POST unterscheiden, kennt SQL-Direct-Clients nicht, doppelt die Splitt-Logik an mehreren Stellen.
2. **PostgREST-Doppel-Instanz.** Verworfen: deckt nicht Direct-DB-Clients oder pg_graphql ab; verdoppelt Container-Anzahl ohne Mehrwert über Supavisor.
3. **Supavisor (chosen).** Einheitlicher Splitt-Punkt; deckt alle DB-Konsumenten ab; Operator-transparent.
4. **Application-Hint-Header.** Verworfen: leaky abstraction, bricht Drop-in-Versprechen.

## Consequences

- **B6**: Cluster-Mode-Implementierung muss Supavisor-Konfig-Generator im studio-backend bauen (Replica-Erweiterung als Studio-Wizard).
- **Replication-Lag-Risiko**: Read-after-Write auf Replica innerhalb Lag-Fenster liefert stale Daten — `_ntts.replication_lag_status` + `aggregate_health()` (F-HC-5) integrieren Lag-Threshold; F-LOG-4 fires WARN bei >5s.
- **Transaction-Mode-Limit**: Supavisor's Transaction-Mode bricht bei Prepared-Statements und Sessions — Doku muss Single-Node-Default-Pfad als "kein read-after-write-Problem" beworben halten.
- **Realtime-Multi-Node** bleibt separat (Phoenix-Cluster, eigene Distrib-Concerns) — dieser ADR adressiert nur den DB-Read-Path.

## Related

- PRD §10 Q6 (resolved by this ADR)
- [ADR-0002](./0002-single-node-default-cluster-optional.md) (single-node default, cluster optional)
- F-POOL-1..3 (Supavisor heute)
- B6 Roadmap (Cluster Mode + Read Replicas)
