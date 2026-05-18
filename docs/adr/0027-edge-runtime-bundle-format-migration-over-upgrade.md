# ADR-0027: Edge-Runtime Bundle-Format-Migration über NTTS-Upgrade

**Status:** Accepted
**Date:** 2026-05-18
**Resolves:** Design-Lücke in ADR-0011 (Function-Bundle-Format-Migration über NTTS-Upgrade)

## Context

Edge Runtime (Deno-V8-isolates, NTTS-Fork) ist eine der wrapped Upstreams (PRD §3.1, ADR-0023). Edge Runtime bumpt regelmäßig das eszip-Bundle-Format — bisweilen als Major-Break (eszip v1 → v2 in Deno 1.x→2.x), bisweilen patch-kompatibel.

NTTS persistiert Function-Bundles als `_ntts.function_bundles.eszip_bytes` (F-FN-2). **Der Operator-App-Source-Code lebt nicht in `_ntts`** — er lebt im App-Repo des Operators (`functions/<name>/index.ts`). NTTS hat keinen lokalen Re-Compile-Pfad ohne diesen Source.

Das bedeutet: eine `ntts upgrade`, die einen Edge-Runtime-Major-Bump mitbringt, kann nicht einfach alle Bundles re-encoden. Drei Optionen wurden in der Grilling-Session 2026-05-18 betrachtet:

## Considered Options

1. **Lazy Re-Bundle bei Invocation.** Edge Runtime versucht das alte Format zu laden, fails, triggert irgendwie Re-Bundle aus dem Source-Code.
   - **Verworfen:** Source-Code ist nicht in `_ntts`. Würde verlangen, dass NTTS bei jedem Deploy auch den TS-Source als bytea persistiert — ein neuer Storage-Layer mit eigenen Privacy- und Größen-Implikationen. Bricht die Bundle-as-immutable-artifact-Eigenschaft (CONTEXT.md: "Bundle ist eszip-compiled, immutable").

2. **Eager Re-Bundle in upgrade-apply.** Upgrade-apply iteriert über alle Bundles, läuft Re-Compile.
   - **Verworfen:** Setzt voraus dass Source verfügbar ist. Nicht der Fall — siehe (1).

3. **Multi-Format-Support im Edge-Runtime-Fork.** Edge Runtime versteht die letzten N eszip-Format-Generationen parallel.
   - **Akzeptiert als Primärpfad** für patch- und minor-Format-Drifts.

4. **Out-of-range Fail-Soft + Operator-Redeploy.** Bei Major-Format-Breaks außerhalb des Multi-Format-Support-Fensters: betroffene Functions in `_ntts.function_deployments` als `status='requires_redeploy'` markieren, NICHT aufrufbar bis Operator `ntts fn deploy` aus seinem App-Repo neu pusht. Upgrade selbst geht durch.
   - **Akzeptiert als Fallbackpfad.**

## Decision

**Dreiteilige Strategie:**

### 1. Schema-Erweiterung von `_ntts.function_bundles`

```sql
ALTER TABLE _ntts.function_bundles
  ADD COLUMN eszip_format_version int NOT NULL DEFAULT 1;
```

Bei jedem Bundle-INSERT (F-FN-2) schreibt die Pipeline die Format-Version, mit der der eszip-Encoder das Artifact erzeugt hat.

### 2. Multi-Format-Support im NTTS-Edge-Runtime-Fork

Edge-Runtime-Fork hält gleichzeitig Decoder/Loader für die letzten **drei** eszip-Format-Generationen (`SUPPORTED_FORMATS: int[]`). Konkrete Versionen werden pro NTTS-Release in `RELEASE_NOTES.md` und in `apps/edge-runtime/SUPPORTED_FORMATS.md` deklariert.

Multi-Format-Pfad wird via Rust-Match auf `eszip_format_version` aus `_ntts.function_bundles` gewählt. Decode-Failure pro Bundle wirft eine typisierte Exception (`UnsupportedEszipFormat`), die nach unten propagiert zu `function_deployments.status = 'unsupported_eszip_format'`.

### 3. Upgrade-Prep-Check + Soft-Fail

In `_internal upgrade-prep` (ADR-0011 Step 4):

```sql
SELECT count(*) FROM _ntts.function_bundles
WHERE eszip_format_version < (SELECT min_supported_eszip_format FROM new_image_metadata);
```

- **Ergebnis = 0:** Upgrade geht through. Multi-Format-Decoder im neuen Image kann alle Bundles laden.
- **Ergebnis > 0:** Upgrade-Prep meldet die Liste betroffener Functions per Studio-Banner und CLI-Output. Operator kann wählen:
  - **`ntts upgrade --allow-stale-bundles`:** Upgrade läuft durch. Affected Functions bekommen `_ntts.function_deployments.status = 'requires_redeploy'`. HTTP-Trigger antworten 503 mit Body `{"error":"function_requires_redeploy","function":"<name>","since":"<ntts_version>"}`. F2F-Calls scheitern mit `FunctionRequiresRedeploy`.
  - **`ntts upgrade --refuse-stale-bundles` (default):** Upgrade-Prep bricht mit Exit 2 ab. Operator muss vor Upgrade aus seinem App-Repo `ntts fn deploy --all` laufen lassen — die regeneriert Bundles im aktuellen Format. Dann Upgrade retry.

### 4. CLI-Augmentation

- `ntts fn list --filter-status requires_redeploy` — Übersicht.
- `ntts fn deploy --all-stale` — bulk-deploy aller `requires_redeploy`-Functions aus dem App-Repo.
- `ntts upgrade --check` zeigt vorab welche Bundles out-of-range wären.

## Consequences

### Operator-UX

Major-Format-Breaks sind selten (alle 2–4 NTTS-Major-Releases). Wenn sie passieren:

- **Upgrade-Pfad:** `ntts upgrade --check` warnt. Operator pusht aus App-Repo `ntts fn deploy --all`. Re-Pipeline läuft, neue Bundles mit aktuellem Format landen in `_ntts.function_bundles`. Operator läuft `ntts upgrade` (default: refuse-stale-bundles). Upgrade clean.
- **Stale-Bundles-Path:** Operator hat keinen Zugriff aufs App-Repo (z.B. Recovery-Szenario nach Festplattenverlust). `ntts upgrade --allow-stale-bundles` lässt das System hochkommen mit `requires_redeploy`-Functions. Sobald Operator wieder am App-Repo ist, deployt er und alles ist normal.

### Implementations-Aufwand

- Schema-Migration für `eszip_format_version` Spalte: trivial, 1 NTTS-Internal-Migration.
- Multi-Format-Decoder im Rust-Fork: ~200–500 LOC pro unterstützter Format-Generation. Drei Generations = ~1.000–1.500 LOC zusätzlicher Fork-Surface. Maintenance-Last hängt von Upstream-Format-Stabilität ab.
- CLI-Erweiterungen (`fn list --filter-status`, `fn deploy --all-stale`): ~1–2 Tage Solo.
- Upgrade-Prep-Check-Logik in studio-backend: ~1 Tag Solo.

### Was die Architektur nicht löst

- **Format-Bumps häufiger als alle drei NTTS-Releases:** wenn Edge Runtime alle 3 Monate eine Major-Format-Break macht, sind selbst 3 unterstützte Generations zu wenig. Mitigation: Edge-Runtime-Fork-Pin (Smallest-Possible-Patch-Prinzip aus Frage 13 Memory) → wir folgen Upstream langsamer, akzeptieren security-lag (30 Tage per Memory-Entscheidung) gegen format-stability.
- **Operator ohne App-Repo-Zugang:** `--allow-stale-bundles` lässt das System laufen, aber affected Functions sind tot bis Redeploy. Kein automatischer Recovery-Pfad — Operator muss handeln.
- **Bundle-Format-Forward-Compat:** kein Versuch alte Bundles "vorwärts zu migrieren". Format-Breaks sind Format-Breaks; Re-Source-Compile ist der einzige saubere Pfad.

## Related

- F-FN-2 (Bundle-Storage in `_ntts.function_bundles`) — wird um `eszip_format_version` Spalte erweitert.
- F-FN-14 (Boot-Hydration) — propagiert `unsupported_eszip_format` als Function-Status statt globalem Boot-Fail.
- ADR-0011 (NTTS-Upgrades) — referenziert diese ADR für Bundle-Format-Migration-Pfad.
- ADR-0023 (TS-first + Caddy-Plugin Go-Exception) — wird durch diese ADR weiter unterminiert; Rust-Fork-Surface wächst um Multi-Format-Decoder-Code.
