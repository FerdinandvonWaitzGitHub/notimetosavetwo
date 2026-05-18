# ADR-0023: Monorepo, TypeScript-first für eigenen Code; Forks in nativen Upstream-Sprachen

**Status:** Accepted (revised 2026-05-18 — siehe Addendum)
**Date:** 2026-05-18

## Context

Die PRD zählt auf, *was* NTTS selbst baut (Control-Plane, Studio-Backend, Studio-Frontend, CLI, ntts-typegen-Worker, Gateway-Plugin, Mobile-SDK-Wrapper), legt aber weder die Repo-Struktur noch die Sprach-/Stack-Wahl pro Komponente fest. Ohne diese Entscheidung kann keine Datei geschrieben werden.

Alternativen, die wir abgewogen haben:

1. **Monorepo, TypeScript-first.** Alle gebauten Komponenten in TS, geteilte Typen via internem Package, ein einziger Release-Schedule. Caddy-Plugin notwendigerweise Go.
2. **Monorepo, gemischter Stack.** CLI in Rust (Single-Binary), Studio Next.js, Backend Node, Gateway Go. Je Komponente die "beste" Sprache, mehr Build-Komplexität.
3. **Polyrepo.** Separate Repos pro Komponente. Maximale Entkopplung, aber Cross-Repo-Type-Sharing teuer, Release-Koordination bricht das Single-Deployment-Versprechen.

## Decision

**Monorepo, TypeScript-first.**

Layout:

```
/
├── apps/
│   ├── studio-backend/        Node + Fastify (Control-Plane + Studio-API)
│   ├── studio-frontend/       Next.js
│   ├── ntts-typegen/          Node-Worker (LISTEN ntts_typegen)
│   └── cli/                   Node, Bundle via esbuild + pkg/bun → Single-Binary
├── gateway/
│   └── ntts-edge/             Go (Caddy-Plugin) — einzige Nicht-TS-Komponente
├── packages/
│   ├── types/                 @ntts/types — generierte DB-Typen, geteilte Domain-Typen
│   ├── sdk-js/                Thin Re-Export @supabase/supabase-js + NTTS-Generics
│   └── pipeline/              Pre-Deploy-Pipeline (8 Stages), geteilt CLI ↔ Studio-Backend
├── mobile-sdks/               Swift / Kotlin / Flutter / Python — eigene Toolchains, im selben Repo
├── docker/                    Dockerfiles + compose-Files
├── migrations/                NTTS-interne Migrationen (`_ntts/migrations/`) — Image-embedded
└── docs/                      ADRs + Operator-Docs
```

Tooling: pnpm Workspaces, Turborepo für Build-Graph, tsup/esbuild für Bundle-Production, Vitest für Tests.

Begründung:
- Single-Deployment-Appliance → ein Release-Schedule → ein Repo.
- CLI, Studio-Backend, Control-Plane, Studio-Frontend teilen die generierten DB-Types — Cross-Package-Type-Sharing ist im Monorepo trivial.
- Edge-Runtime ist V8 → TS überall reduziert die kognitive Last und den Build-Maschinen-Park.
- Gateway-Plugin bleibt Go, weil Caddy in Go geschrieben ist.

## Consequences

**Positiv**
- Eine `pnpm install` reicht für die gesamte Codebasis (außer Mobile-SDKs).
- Type-Drift zwischen Komponenten unmöglich — alle ziehen aus `@ntts/types`.
- Atomare Cross-Component-PRs.

**Negativ**
- CLI-Verteilung als Single-Binary erfordert pkg/bun-Toolchain — leichte Build-Komplexität.
- Go-Plugin-Build steht außerhalb des pnpm-Graphs — separater CI-Step.
- Mobile-SDKs koexistieren als eigenständige Toolchain-Inseln; CI muss das tolerieren.

## Related

- Keine Vorgänger-ADR.
- Berührt zukünftige ADRs zu Release-Pipeline, Image-Build, CLI-Distribution.

---

## Addendum 2026-05-18 — Forks als zweite Sprach-Achse

Die ursprüngliche Entscheidung "TypeScript-first; Caddy-Plugin als Go-Ausnahme" war für **eigene NTTS-Komponenten** korrekt und bleibt es. Was die ursprüngliche Formulierung *nicht* abdeckte: die Strategie für die wrapped Upstreams. In der Grilling-Session am 2026-05-18 wurde Option (a) gewählt — vier Upstream-Komponenten werden als **Forks** maintained, nicht als reine Wraps. Damit kommen drei weitere Primärsprachen hinzu.

### Aktualisierte Sprach-Matrix

| Codebasis-Kategorie | Sprache | Beispiele |
|---|---|---|
| Eigene NTTS-Komponenten | **TypeScript** | studio-backend, studio-frontend, ntts-typegen, CLI, JS-SDK, packages/* |
| Caddy-Plugin (eigene Komponente) | **Go** | gateway/ntts-edge/ |
| Fork: Edge Runtime | **Rust** | forks/edge-runtime/ — F-FN-18, F-FN-8, F-FN-6 patches |
| Fork: supabase-realtime | **Elixir** | forks/supabase-realtime/ — F-RT-6 HMAC trust, ADR-0026 three-layer authz, multi-node distribution |
| Fork: Supavisor | **Elixir** | forks/supavisor/ — ADR-0025 statement-inspection r/w splitting |
| Fork: GoTrue | **Go** | forks/gotrue/ — F-AUTH-9 multi-IdP routing, F-AUTH-12 NOTIFY ntts_jwt_changed |
| Mobile-SDK (v1.0) | **Swift** | mobile-sdks/swift/ — Kotlin/Flutter/Python deferred post-v1.0 |
| Database-Side | **SQL + PL/pgSQL** | migrations/, _ntts/migrations/ |

### Aktualisiertes Repo-Layout

```
/
├── apps/
│   ├── studio-backend/        Node + Fastify (Control-Plane + Studio-API)
│   ├── studio-frontend/       Next.js
│   ├── ntts-typegen/          Node-Worker (LISTEN ntts_typegen)
│   └── cli/                   Node, Bundle via esbuild + bun --compile → Single-Binary
├── gateway/
│   └── ntts-edge/             Go (Caddy-Plugin) — eigene Komponente
├── forks/                     ← NEU: gepatchte Upstream-Trees
│   ├── edge-runtime/          Rust — F-FN-18, F-FN-8, F-FN-6 patches
│   ├── supabase-realtime/     Elixir — F-RT-6 HMAC, ADR-0026 three-layer authz, multi-node
│   ├── supavisor/             Elixir — ADR-0025 r/w split via statement-inspection
│   └── gotrue/                Go — F-AUTH-9 multi-IdP, F-AUTH-12 NOTIFY ntts_jwt_changed
├── packages/
│   ├── types/                 @ntts/types
│   ├── sdk-js/                Thin Re-Export @supabase/supabase-js + NTTS-Generics
│   └── pipeline/              Pre-Deploy-Pipeline (8 Stages), geteilt CLI ↔ Studio-Backend
├── mobile-sdks/
│   └── swift/                 Swift — v1.0 Mobile-SDK
│   (kotlin/flutter/python deferred post-v1.0)
├── docker/                    Dockerfiles + compose-Files
├── migrations/                NTTS-interne Migrationen — Image-embedded
└── docs/                      ADRs + Operator-Docs
```

### Konsequenzen des Fork-Modells

- **Fork-Maintenance-Cadence:** ~2–4 Wochen pro Quartal kumulativ für Drift-Merging gegen Upstream. Security-Releases mit 30-Tage-Lag akzeptiert (Grilling-Session-Entscheidung — solo, no external-operator-base bisher).
- **Lernkurve:** Ferdinand hat keine production-grade-Vorerfahrung in Rust, Elixir, oder Go. Lernen-while-Building akzeptiert. Empfohlene Bring-up-Reihenfolge: **Go (Caddy-Plugin + GoTrue) → Elixir (Realtime + Supavisor) → Rust (Edge Runtime)**. Easier-Sprachen zuerst, Confidence aufbauen, dann das härteste Stück (unsafe-Rust + V8-Boundary) zuletzt.
- **Smallest-Possible-Patch-Prinzip:** Jeder Fork-Patch muss minimal sein. Beispielsweise: F-FN-18 (`/_internal/invoke`) könnte alternativ in studio-backend als TS-Sidecar leben statt als Rust-Patch — Trade-off zwischen Trust-Surface und Sprachen-Surface bei jedem Patch ehrlich abwägen.
- **CI-Toolchain:** Jeder Fork bringt seine eigene Build-Toolchain mit: cargo (Rust), mix (Elixir), go build, plus die TS-Toolchain. CI-Build-Zeit ~30+ Minuten ist realistisch. Multi-Stage-Cache-Strategie + Conditional-Path-Triggers (nur Forks die geänderte Pfade haben rebuilden) ist Pflicht.
- **Externe Security-Audit der Fork-Patches** vor v1.0-Ship — die eine Stelle wo Solo-Bauweise einen echten externen Audit-Kosten erfordert. Budgetiert in Grilling-Session-Entscheidung zu F-RT-4.

### Was unverändert bleibt

- Monorepo-Struktur ist weiter richtig: alle eigenen Komponenten + alle Forks in einem Repo, ein Release-Schedule, geteilte `@ntts/types`.
- pnpm-Workspaces + Turborepo für den TS-Teil — Forks haben ihre eigenen Build-Steps außerhalb des pnpm-Graphs (separate CI-Jobs).
- "TypeScript-first" gilt weiter für eigene Komponenten. Fremder Code wird in seiner nativen Sprache als Fork gehalten — wir refraktorieren Edge-Runtime nicht zu TypeScript.
