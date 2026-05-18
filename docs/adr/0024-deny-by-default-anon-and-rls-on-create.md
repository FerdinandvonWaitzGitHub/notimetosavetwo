# ADR-0024: Deny-by-default für `anon`, RLS-on-create für `public.*`

**Status:** Accepted
**Date:** 2026-05-18
**Resolves:** PRD §10, Open Question 1

## Context

Supabase legt neu erstellte Tabellen in `public.*` ohne RLS an. `anon` und `authenticated` haben damit Lese-/Schreibrechte, sobald das Schema reload-cached ist. Dieser Default ist als bekannte Foot-Gun dokumentiert — Operator vergisst RLS einzuschalten → API-Endpoint liefert die ganze Tabelle an jeden mit `anon`-Key.

NTTS will *Drop-in für Supabase am SDK-Surface* sein, aber wir sind nicht an die operativen Defaults gebunden. Sicherheits-Default kann härter sein, ohne die SDK-Kompatibilität zu brechen — eine Supabase-App, die auf öffentlichen Tabellen lief, wird beim Import auf NTTS einfach 0 Rows zurückbekommen, bis Policies geschrieben sind. Lautes Failen ist besser als stilles Leaken.

## Decision

**Drei Garantien:**

1. **`_ntts.*` ist hartkodiert versperrt.** `anon` und `authenticated` erhalten *niemals* GRANTs auf irgendetwas unter `_ntts`. Migrations-Linter im studio-backend lehnt `GRANT ... TO anon` auf `_ntts.*` ab.
2. **`public.*` wird RLS-on-create angelegt.** Studio-Wizard, `ntts db make`-Template, und Migrations-Linter setzen für jede neue `CREATE TABLE public.<name>`:
   - `ALTER TABLE public.<name> ENABLE ROW LEVEL SECURITY;`
   - keine Policies (also: alles deny, bis Operator Policy hinzufügt)
   - Stub-Kommentar im Migration-File, der zur ersten Policy auffordert
3. **`ntts db verify` warnt** bei jeder `public.*`-Tabelle ohne RLS — Soft-Warn (nicht Hard-Block), weil Operator ja absichtlich Service-Tabellen ohne RLS bauen kann (z. B. `public.config` mit `REVOKE` von allen App-Roles und nur Server-seitigem Zugriff).

Wrapped-Service-Schemas (`auth.*`, `storage.*`, `realtime.*`) behalten ihre GoTrue/Storage/Realtime-Defaults — die wissen selbst, was sie tun und sind durch ihren eigenen Code geschützt.

## Considered Options

1. **Deny-by-default + RLS-on-create (chosen).** Lautes Failen statt stilles Leaken.
2. **Deny-by-default, RLS optional.** Anon hat keine GRANTs, aber Tabellen ohne RLS. Operator muss zwei Schritte machen statt einen — mehr Default-Holes.
3. **Supabase-Default (öffentlich bis RLS).** Drop-in im operativen Sinne, aber wir kopieren bewusst nicht den bekannten Foot-Gun.

## Consequences

- Supabase-Apps, die ohne RLS-Policies migriert werden, returnieren initial 0 Rows. Migrations-Doku muss das deutlich machen.
- Der "Create Table"-Wizard hat einen zwangsbegleitenden Policy-Editor-Step (kann übersprungen werden, aber Default ist "jetzt Policy schreiben").
- `ntts db verify` liefert non-zero exit für RLS-Warnungen *nicht* — das wäre zu aggressiv für legitime Server-Tabellen.
- Der `_ntts`-Lockdown braucht einen Migrations-Linter, der DDL-Statements parst — das ist neue Arbeit, gehört in `apps/studio-backend/src/migrations/linter.ts`.

## Related

- PRD §10 Q1 (resolved by this ADR)
- F-REST-5 (RLS on all paths)
- F-MIG-1..10 (Migrations)
- Berührt zukünftiges F-DB-7 (Default-Privilegien-Garantien).
