# PRD-Backend — NoTimeToSave Backend

**Status:** v0.4 full-parity cut
**Date:** 2026-05-18
**Author:** Ferdinand von Waitz

> Vocabulary is canonical per [CONTEXT.md](./CONTEXT.md). Architectural decisions in [docs/adr/](./docs/adr/).

---

## 1. Mission

Self-hosted EU-sovereign single-deployment BaaS appliance. Full Supabase feature parity at the SDK surface: Postgres + REST + GraphQL + Pooler + Auth + Storage + Realtime + Functions + Vault + Queues. Drop-in for Supabase per [CONTEXT.md](./CONTEXT.md). Existing SDK code runs unchanged.

Strategy: wrap battle-tested OSS (PostgREST, pg_graphql, Supavisor, GoTrue, Edge Runtime, supabase-storage, supabase-realtime). Build only where value is: control-plane, studio, pre-deploy pipeline, DX, mobile SDKs.

---

## 2. Why

- Makers need backend, no vendor lock-in.
- Supabase Cloud = US law, expensive, locked.
- EU sovereign, GDPR clean, own infra.
- Single Deployment by design. Multi-Operator within one Deployment supported.

---

## 3. Vision

> `docker compose up`. Stack runs. Setup Token printed. First Operator claims admin. Point app from `xyz.supabase.co` to own domain. Works.

### Principles

- Compat over invention. Supabase shape wins.
- Wrap before reimplement.
- Postgres-first. Hard consistency in DB. Logic in TS Functions. Postgres is also control-plane bus.
- Single-Deployment by design. Multi-Operator within one Deployment.
- App-backend appliance, single-node default. Cluster Mode (read replicas, multi-node Realtime) optional. See [ADR-0002](./docs/adr/0002-single-node-default-cluster-optional.md).
- **Apache License 2.0** for NTTS-distributed code (explicit patent grant; aligns with Supabase main repo). Postgres extensions exempt from the permissive-only rule — loaded at runtime, not redistributed. See [ADR-0001](./docs/adr/0001-license-policy-extensions-exempt.md).

### 3.1 Build vs Wrap vs Fork

Four categories: **Use** (upstream artifact loaded at runtime, no NTTS code involved), **Wrap** (upstream container with config-only NTTS integration, no upstream-tree patches), **Fork** (NTTS maintains a patched upstream tree synced against upstream — language and maintenance implications per [ADR-0023](./docs/adr/0023-monorepo-typescript-first-with-caddy-plugin-as-go-exception.md) Addendum), **Build** (own NTTS code).

| Component | Strategy |
|---|---|
| Postgres 16 | Use |
| pg_graphql | Use (Postgres extension) |
| PostgREST | Wrap (JWT-verify off via config) |
| supabase-storage | Wrap (JWT-verify off via config) |
| Caddy (base for `ntts-edge` Gateway) | Wrap |
| WAL-G (PITR backup) | Wrap |
| GoTrue | **Fork (Go)** — F-AUTH-9 multi-IdP routing, F-AUTH-12 `NOTIFY ntts_jwt_changed` |
| Edge Runtime (V8 isolates) | **Fork (Rust)** — F-FN-18 `/_internal/invoke`, F-FN-8 signature-verify, F-FN-6 per-execution metrics |
| supabase-realtime (Phoenix) | **Fork (Elixir)** — F-RT-6 HMAC trust, [ADR-0026](./docs/adr/0026-realtime-three-layer-authorization-bucketing-snapshot-cache-bulk-eval.md) three-layer authorization, multi-node distribution |
| Supavisor (pooler) | **Fork (Elixir)** — [ADR-0025](./docs/adr/0025-cluster-mode-supavisor-as-readwrite-splitter.md) statement-inspection r/w splitting, sticky-write |
| Gateway (`ntts-edge`: JWT, rate-limit, request-ID, service-role) | Build (Go-plugin on Caddy) |
| Control-plane / studio-backend | Build (TS/Node) |
| Studio frontend | Build (TS/Next.js) |
| Pre-deploy Pipeline | Build (TS) |
| Versioning + rollback | Build (TS) |
| Schema-compat-check | Build (TS) |
| Secret-injection (pgcrypto JIT) | Build (SQL + TS) |
| Studio-user management + Setup Token bootstrap | Build (TS) |
| AI-assistant adapter (provider-pluggable) | Build (TS) |
| CLI `ntts` | Build (TS, `bun --compile`) |
| JS SDK | Build (thin re-export of `@supabase/supabase-js`) |
| Mobile SDK (Swift, v1.0) | Build (thin re-export + type-gen) — Kotlin/Flutter/Python deferred post-v1.0 |

### 3.2 Architecture

- **Single Gateway (`ntts-edge`)** is the only externally reachable component. JWT validated once at the Gateway; upstreams trust internal headers. See [ADR-0004](./docs/adr/0004-single-gateway-jwt-validated-once.md).
- Postgres is control-plane bus. All studio state in `_ntts` schema. NOTIFY/LISTEN for invalidation. No separate config store.
- Atomic deploys = one transaction (INSERT Bundle, UPDATE pointer, NOTIFY).
- Backup: `pg_dump` snapshot **or** WAL-G PITR. Operator picks per Deployment.
- Single-node default. Cluster Mode optional.
- Docker image variants: `ntts-postgres:slim` (MIT-pure extensions) vs `ntts-postgres:full` (adds PostGIS, pgroonga, plv8, plpython).
- Rate-limit state lives in Postgres (`_ntts.rate_limit_buckets`) — no Redis container required.

---

## 4. Scope

Everything in this section is in scope. No phasing.

### 4.1 Database

| Feature |
|---|
| Postgres 16 dedicated |
| REST via PostgREST wrap |
| Full PostgREST filter DSL, FK embed, bulk, Range header |
| RPC via PostgREST |
| GraphQL via pg_graphql (`/graphql/v1`) |
| Connection Pooler (Supavisor) at `:6543`, transaction + session mode |
| RLS |
| Migrations (forward-only, hash, see [ADR-0010](./docs/adr/0010-migrations-forward-only-files-in-repo.md)) |
| Webhooks via pg_net |
| **Slim extensions** (default): pgcrypto, uuid-ossp, pg_stat_statements, pg_cron, pg_net, pgvector, pg_jsonschema, pgmq, pgaudit, pgsodium, wrappers, postgres_fdw, pg_graphql, supavisor |
| **Full-image extensions** (add-on): PostGIS, pgroonga, plv8, plpython |
| Schema-cache reload via `NOTIFY pgrst` (pg_graphql piggybacks) |
| Read Replicas (Cluster Mode, optional) |
| PITR backup via WAL-G |
| Vault (pgsodium-encrypted secrets for SQL-tier consumers — distinct from Function Secrets, see [ADR-0008](./docs/adr/0008-three-classes-of-secrets.md)) |

### 4.2 Auth (GoTrue wrap)

| Feature |
|---|
| Email + password + verify + reset |
| Phone-OTP, Email-OTP |
| Magic link |
| All GoTrue OAuth providers (Google, Apple, GitHub, Discord, Slack, Microsoft, Bitbucket, LinkedIn, GitLab, Notion, Figma, Twitter, Twitch, Facebook, Spotify, Kakao, Keycloak, AzureAD, Zoom, Fly, WorkOS, generic OIDC) |
| SSO / SAML 2.0 |
| Anonymous sign-in |
| MFA TOTP |
| MFA SMS (catalog-driven SMS provider — Twilio / Twilio Verify / MessageBird / Vonage / TextLocal via F-AUTH-11) |
| JWT + session revoke |
| Auth hooks (before/after sign-in, custom JWT claims) — implemented as Edge Function calls |
| Custom SMTP (operator-supplied) |
| Custom email templates (i18n + branding) |
| Rate-limit (NTTS gateway) |

### 4.3 Storage (supabase-storage wrap)

| Feature |
|---|
| Buckets, multipart, signed URLs |
| Per-bucket constraints |
| RLS on objects |
| Local FS backend |
| S3 backend |
| S3-compatible API (third-party S3 SDKs work against Storage) |
| TUS resumable uploads |
| Image transformations |
| CDN passthrough (`Cache-Control` + origin shielding) |

### 4.4 Realtime (supabase-realtime wrap)

| Feature |
|---|
| Channels |
| Postgres changes (logical replication) |
| Presence |
| Broadcast |
| RLS-filter |
| Multi-node (Cluster Mode, optional) |

### 4.5 Logic-Tier (Edge Functions)

**Architecture:** wrap Supabase Edge Runtime (MIT, Rust, V8 user-worker isolates). One container, fixed. Concurrency = V8 isolates per invocation. Bundles + deployment state in `_ntts`. Invalidation via LISTEN/NOTIFY. Rollback = pointer-UPDATE. Pipeline + differentiation logic sit in NTTS control-plane *before* the runtime, not inside it.

| Feature |
|---|
| Edge Runtime wrap (V8 isolates) |
| Visibility public / private / event |
| Bundles in `_ntts.function_bundles` |
| Pointer in `_ntts.function_deployments` |
| LISTEN/NOTIFY cache invalidation |
| Boot hydration from `_ntts` |
| HTTP trigger `/functions/v1/:name` (public only) |
| Function-to-function call (incl. private) |
| DB-event trigger (pg_net) |
| Cron trigger (pg_cron) |
| Typed DB client in Function (RLS-aware) |
| Per-Function secrets (pgcrypto, JIT) |
| Pre-deploy Pipeline (8 stages) |
| Versioning + 1-click rollback (last 20 Bundles per Function) |
| Schema-compat-check (bidirectional) |
| Sandbox quotas (CPU / mem / duration / egress) |
| Logs with secret-scrubbing → `_ntts.function_logs` |
| Audit log (override reason required) |
| OTel tracing |
| Background tasks (`waitUntil`) |
| Bundle storage in S3 (optional, default Postgres `bytea`) |

### 4.6 Studio

| Feature |
|---|
| Task-oriented home |
| Wizards (table, RLS, Function, bucket, cron, App-User-Invite) |
| Operator-Invite (admin-only, separate from App-User-Invite) |
| Inline glossary (Cmd+G) |
| Logic-tier overview |
| Editor: live TS check + schema-aware types |
| Pre-deploy Pipeline live UI |
| Apple-HIG-Ästhetik + WCAG 2.1 AA. Stack: shadcn/ui (Radix + Tailwind), copy-paste-Components in `apps/studio-frontend/src/ui/`, Design-Tokens in `apps/studio-frontend/src/tokens/`. SF-Pro-Display primär, Inter Fallback. Light + Dark Mode |
| German UI default, English code/CLI/logs. i18next + ICU MessageFormat. MVP-Locales DE+EN; weitere community-contrib post-v1. Auth-Email-Templates per-locale in `_ntts.auth_templates`, Operator-extensible. Details in §4.11 |
| RLS test-sandbox with personas |
| Visual schema designer (ERD) |
| AI assistant (default-off, provider-pluggable — see [ADR-0003](./docs/adr/0003-ai-assistant-provider-agnostic-default-off.md)) |
| Advisor (security + performance) |
| Log explorer / reports |
| Auto-generated API docs |
| Backups UI (`pg_dump` + PITR) |
| Studio-user management (admin only — invite, role, MFA enrol, disable) |

### 4.7 Control Plane

Node/TS service in the same compose. State in `_ntts`. Docker-socket for service lifecycle. No K8s, no Nomad.

| Feature |
|---|
| Studio-backend API |
| Docker-socket integration |
| `_ntts` schema management |
| NOTIFY emitter on state change |
| Audit log (immutable, write actions): `_ntts.studio_audit_write` |
| Audit log (PII reads by Operators): `_ntts.studio_audit_read` |
| Studio-user table `_ntts.studio_users` |
| Setup Token bootstrap (printed to stdout on fresh start, expires 24h) |
| Operator-Invite issuance (signed link, expires 24h) |

### 4.7a Release & CI Pipeline

**CI-Stack:** GitHub Actions. **Image-Registry:** GHCR primär (`ghcr.io/ntts/*`), Docker Hub als Sekundär-Mirror. **6 Component-Images:** `ntts-edge`, `ntts-studio-backend`, `ntts-typegen`, `ntts-egress-proxy`, `ntts-postgres:slim`, `ntts-postgres:full`. **Tag-Schema:** `:v1.3.2` (patch), `:v1.3` (minor-track, auto-updated), `:latest` (latest stable), `:edge` (main-branch latest commit). **Multi-Arch:** linux/amd64 + linux/arm64 via Docker buildx (Apple Silicon dev, AWS Graviton, Hetzner ARM-VPS). **Versionierung:** [changesets](https://github.com/changesets/changesets) für Monorepo-Internal-Package-Versions; [release-please](https://github.com/googleapis/release-please) für Release-PR-Automation. **Cadence:** Minor monatlich, Patch on-demand, `:edge` on jeden main-merge. **Compose-File:** `docker/compose.yml` mit `${NTTS_VERSION:-latest}`-pinning. **Image-Signing:** Cosign/sigstore-Signaturen pro Image; `ntts upgrade` verifiziert Signaturen vor Pull (F-UPG-3 erweitert). Mobile-SDK-Publish lockstep mit Minor-Release (F-SDK-5).

### 4.8 Platform & DevEx

| Feature |
|---|
| CLI `ntts` (full coverage of state-mutation Studio operations — see §5.10) |
| Local stack via docker compose |
| TS type-gen from live schema |
| Bundle scanner (`ntts fn scan` — license, size, static-secrets, SBOM) |
| Custom domains + auto-TLS (Caddy) |
| Prometheus metrics |
| Log drains |
| OTel exporter (configurable OTLP endpoint) |

### 4.8a Catalog Schemas

Shared meta-schema (`packages/catalog-schema/`, Zod) treibt drei Catalog-Dateien im Image: `wrappers.catalog.json`, `oauth-providers.catalog.json`, `sms-providers.catalog.json`. Jede Datei trägt `schema_version: int`. Validation an drei Punkten: CI-Build, studio-backend boot (refuse-to-start bei Mismatch), Operator-Reload via `NOTIFY ntts_catalog_reload`. Operator-Extensions via `/etc/ntts/catalogs.d/*.json` (selbe Schemas, gemergt bei Boot) — erlaubt firmen-eigene IdPs/Wrappers/SMS-Treiber ohne NTTS-Fork. Schema-Versionierung mit NTTS-Release: Boot lehnt unbekannte Versionen ab mit scharfem Hinweis "NTTS-Upgrade nötig".

### 4.9 SDK

Thin re-exports of upstream SDKs + NTTS type generics.

| Feature |
|---|
| Drop-in for `@supabase/supabase-js` |
| Generated types |
| Mobile SDKs: Swift, Kotlin, Flutter, Python (re-exports of `supabase-swift`, `supabase-kt`, `supabase-flutter`, `supabase-py`) |

---

## 5. Functional Requirements

### 4.10 Interne HMAC-Schlüsselverwaltung

Zwei HMAC-Keys schützen zwei Trust-Edges, die nicht ausschließlich auf Docker-Network-Trust (ADR-0004) verlassen können:

| Trust-Edge | Key | Wer benötigt |
|---|---|---|
| F2F-Caller → `edge-runtime:9000/_internal/invoke` (F-FN-18) | `hmac_f2f` | Edge-Runtime (alle Worker, callers + listener) |
| ntts-edge `X-NTTS-Auth` → supabase-realtime (F-RT-6) | `hmac_gateway_realtime` | ntts-edge, supabase-realtime |

Storage: `_ntts.platform_secrets[hmac_f2f]` + `_ntts.platform_secrets[hmac_gateway_realtime]`, pgcrypto-encrypted mit `NTTS_MASTER_KEY`. Bootstrap: studio-backend generiert beim ersten `up` zwei 256-bit CSPRNG-Keys. Materialisierung: studio-backend rendert Klar-Keys nach `/var/run/ntts/hmac/{f2f,gateway-realtime}.key` (tmpfs, mode `0400`), Verbraucher-Container mounten dieses Verzeichnis read-only. Hot-Reload: `NOTIFY ntts_hmac_rotated` → studio-backend re-materialisiert → Container mit Inotify-Watch laden neu.

Rotation: `ntts admin rotate-internal-hmac --kind {f2f|gateway-realtime}` (MFA + Pflicht-Reason + 30s Cool-down). Dual-key-verify-Fenster 60s (alt + neu beide akzeptiert), dann alter Key aus `platform_secrets` gelöscht. **Auto-Rotation** beider Keys als letzter Step jeder `ntts upgrade` (F-UPG-3 erweitert).

Andere Gateway↔Upstream-Edges (PostgREST, GoTrue, Storage, Edge-Runtime-public-path) bleiben auf Docker-Network-Trust per ADR-0004 — kein HMAC im MVP, kann später nachgezogen werden falls Multi-Node-Cluster eingeführt.

### 4.11 i18n

- **MVP-Locales:** DE (default) + EN. Weitere FR/ES/IT/… post-v1 als community-Catalog-Contrib, kein Release-Block.
- **Studio-Frontend:** i18next mit ICU MessageFormat. JSON-Catalogs in `apps/studio-frontend/src/i18n/{de,en}/*.json`. Default-Locale via Browser-Header oder Operator-Preference auf `_ntts.studio_users.locale`.
- **Source-of-Truth:** EN-Strings im Code; DE professionell übersetzt; Catalog-Sync via `i18next-parser` + Translator-Workflow (Lokalise oder Crowdin, OOR via PR).
- **Auth-Email-Templates** (F-AUTH-6): DB-stored per-locale, Operator kann beliebig viele Locales nachpflegen ohne NTTS-Rebuild.
- **CLI + Logs + Audit-Strings + Error-Messages: nur EN.** Begründung: CI-Skripte grippen Output; CLI-Output muss locale-stabil sein.
- **Schrift + Numerik:** ICU-MessageFormat handhabt Pluralis, Dates (`{date, date, long}` → "18. Mai 2026" vs "May 18, 2026"), Numbers.

### 5.0 Gateway (`ntts-edge`)

Caddy-based, custom plugin layer. Single externally reachable component. Everything else lives on the internal Docker network `ntts-internal`, not exposed to the host.

- **F-GW-1:** TLS termination + auto-TLS (Let's Encrypt) for custom domains. Certs in volume `/data/caddy`. Cert lifecycle per [ADR-0017](./docs/adr/0017-tls-cert-lifecycle-snapshot-included-monitored-byo-supported.md):
  - **Backup:** `ntts backup snapshot` tars `/data/caddy` alongside `pg_dump`; restore brings certs back without re-issuance (avoids LE rate-limit).
  - **Monitoring:** pg_cron `_ntts.check_cert_expiry()` hourly populates `_ntts.tls_cert_status`; F-LOG-4 alerts at WARN 14d / ERROR 7d / CRITICAL 2d before expiry. Renewal failures → `_ntts.tls_renewal_errors` + ERROR.
  - **BYO-Cert:** per-domain toggle LE/custom; custom key+cert stored in Vault, materialised to tmpfs for Caddy. Operator-managed renewal.
  - **ACME challenge:** HTTP-01 default; DNS-01 opt-in per domain with provider catalog (Cloudflare, Route53, DO, Hetzner, …), API tokens in Vault.
  - **CLI:** `ntts tls list/set/renew`, `ntts tls challenge set`.
- **F-GW-2:** JWT validated **once** at the Gateway. Upstreams (PostgREST, GoTrue, Storage, Realtime, Edge Runtime, Studio backend) have JWT validation disabled and trust internal headers.
- **F-GW-3:** Internal identity headers set by Gateway, never accepted from external clients:
  - `X-NTTS-User-Id` — App User id from `sub` claim
  - `X-NTTS-Role` — `anon` / `authenticated` / `service_role` / `studio_admin` / `studio_developer` / `studio_viewer`
  - `X-NTTS-Request-Id` — UUIDv7, generated if absent
  - `X-NTTS-Operator-Id` — studio_user id for studio routes
- **F-GW-4:** Request-ID propagated to all upstream calls and into `_ntts.function_logs.request_id`.
- **F-GW-5:** Service-role detection: JWT with `role: service_role`, validated against secret known only to Gateway and Operator's env. Sets `X-NTTS-Role: service_role`.
- **F-GW-6:** Rate-limit: sliding-window per (route × identity), counters in `_ntts.rate_limit_buckets`. Default: 60 RPM per IP for unauthenticated routes, 600 RPM per App User for authenticated, no limit for service_role. Operator can override per route.
- **F-GW-7:** Audit-hook on studio routes (`/studio/api/...`): writes `_ntts.studio_audit_write` / `_ntts.studio_audit_read` rows based on route metadata.
- **F-GW-8:** All upstreams configured to listen only on `ntts-internal` network. Direct host-port access blocked.
- **F-GW-9:** Gateway is the only container with public ports (80, 443). Postgres `5432` and Supavisor `6543` are exposed only if operator opts in via env (`NTTS_EXPOSE_POSTGRES=true`).

### 5.1 REST + GraphQL (`/rest/v1`, `/graphql/v1`)

PostgREST and pg_graphql upstream. Auth + rate-limit + request-ID handled by the Gateway (§5.0).

- **F-REST-1:** Full filter DSL incl. `or=(a.eq.1,b.eq.2)` — PostgREST native.
- **F-REST-2:** `select` with FK-embed — PostgREST native.
- **F-REST-3:** Bulk + `Prefer: return=representation` — PostgREST native.
- **F-REST-4:** Range-header pagination — PostgREST native.
- **F-REST-5:** RLS on all paths. Service-role bypass only on service-JWT.
- **F-REST-6:** PostgREST error body format (`code`, `message`, `details`, `hint`).
- **F-REST-7:** Control-plane fires `NOTIFY pgrst, 'reload schema'` after DDL. Migration-path verify-and-wait + SDK retry backstop + documented non-migration window per [ADR-0021](./docs/adr/0021-postgrest-schema-cache-migration-sync-with-sdk-retry-backstop.md). Control-plane polls `/openapi` every 50ms up to 5s after NOTIFY; `ntts db migrate` returns only after verification. Non-migration DDL (Function-code, raw psql via Backstop) has a ≤2s stale-cache window. SDK retries PGRST200-class errors with backoff (100ms, 300ms, 900ms; max 3). Events logged to `_ntts.schema_cache_events`; WARN alert if >5% of trailing-24h events exceed `latency_ms=3000`.
- **F-GQL-1:** `pg_graphql` reachable at `/graphql/v1`, RLS-respecting.
- **F-GQL-2:** GraphQL introspection enabled in dev, disabled in prod by default.

### 5.2 Pooler (`:6543`)

- **F-POOL-1:** Supavisor in compose, reachable at `:6543`, transaction + session mode.
- **F-POOL-2:** Same auth path as direct connections.
- **F-POOL-3:** Operator can scale Supavisor independently of Postgres.

### 5.3 Auth (`/auth/v1`)

GoTrue upstream. Supabase-compatible endpoints.

- **F-AUTH-1:** JWT claims `aud`, `exp`, `sub`, `email`, `role`, `app_metadata`, `user_metadata`.
- **F-AUTH-2:** Phone-OTP 6 digits, 5 min, max 3 verify attempts.
- **F-AUTH-3:** Magic link 15 min, single-use.
- **F-AUTH-4:** Rate-limit in NTTS gateway.
- **F-AUTH-5:** Custom SMTP via env-vars; password-reset emails blocked until configured.
- **F-AUTH-6:** Email templates per locale stored in `_ntts.auth_templates` (default DE + EN, operator can add). **Engine: Go `text/template`** (GoTrue native — Auth-Mails laufen durch GoTrue). Studio-Editor zeigt Go-template-Syntax mit Variablen-Doku. NTTS-interne Mails (Operator-Invite, Health-Alerts, Backup-Reports) verwenden **Eta** im studio-backend (in `_ntts.ntts_email_templates`, separate Tabelle wegen anderer Syntax). MJML/React-Email vertagt auf post-v1.
- **F-AUTH-7:** Auth hooks fire HTTP calls to a designated Function (configured per hook type).
- **F-AUTH-8:** Anonymous sign-in returns JWT with `aud: anon`. RLS policies can target `auth.role() = 'anon'`.
- **F-AUTH-9:** SAML 2.0 via GoTrue SSO config; **multi-IdP** per Deployment mit Email-Domain-Affinity-Routing. Studio-UI listet alle konfigurierten IdPs in `_ntts.auth_saml_providers (id, metadata_xml_or_url, email_domain_pattern, display_name, enabled_at)`. Login-Surface zeigt initial nur Email-Eingabe; nach Eingabe routet GoTrue auf den IdP, dessen `email_domain_pattern` matcht. Fallback bei mehreren Matches: Operator priorisiert via `priority`-Feld. CLI: `ntts auth saml add/list/test/disable/delete`.
- **F-AUTH-10:** All GoTrue OAuth providers Studio-managed via catalog-driven UI per [ADR-0014](./docs/adr/0014-oauth-providers-studio-managed-catalog-driven-with-iac-export.md):
  - **Storage:** `_ntts.auth_oauth_providers` row per provider; client_secrets pgcrypto-encrypted (studio_secrets class per ADR-0008).
  - **Injection:** Studio-backend renders `/var/run/ntts/gotrue.env` on boot and `NOTIFY ntts_oauth_change`; GoTrue reloads.
  - **Catalog:** versioned JSON manifest in NTTS image describes each built-in provider's fields, labels, validation, gotrue_env_mapping; Studio renders forms dynamically.
  - **Custom providers:** Operator can clone `generic_oidc` or `saml` adapter with custom display_name/scopes/help (cannot define arbitrary new OAuth2 providers — GoTrue has no generic OAuth2 adapter).
  - **Validation:** static (schema + OIDC discovery ping) plus admin-MFA-gated Test-Sign-In flow that completes real OAuth dance against side-effect-free callback; results in `_ntts.auth_oauth_test_log`.
  - **Rotation:** two-field UI, secret-version bump, NOTIFY → GoTrue reload; NTTS-issued user sessions stay valid.
  - **Role gates:** edit/reveal/test/disable/delete = admin + MFA re-auth; view-without-secrets = all Operators.
  - **Disable** (default): keep row, drop `_ENABLED=true`. **Delete:** destructive, MFA + name-confirm, audit snapshot.
  - **IaC export/import:** `ntts auth oauth export > auth.oauth.toml` (secrets externalised), `ntts auth oauth apply -f`.
- **F-AUTH-11:** SMS-provider auswahl via catalog (analog F-AUTH-10):
  - **Catalog:** `/etc/ntts/sms-providers.catalog.json` listet GoTrue's native providers (Twilio, Twilio Verify, MessageBird, Vonage, TextLocal). Studio rendert Setup-Formular dynamisch aus dem Schema.
  - **EU-Sovereignty-Empfehlung:** Studio markiert MessageBird (NL) und Vonage (NO/UK) als "EU-recommended"; Twilio bleibt verfügbar mit US-Hinweis. TextLocal markiert als "India-only".
  - **Storage:** `_ntts.auth_sms_provider (provider_id, config_json, secrets_vault_ref, enabled_at)`; credentials pgcrypto-encrypted (studio_secrets class).
  - **Reload:** Studio-Backend rendert `/var/run/ntts/gotrue.env` neu, `NOTIFY ntts_sms_change` → GoTrue-Reload (gleiche Pipeline wie OAuth-Reload in F-AUTH-10).
  - **CLI:** `ntts auth sms set <provider>`, `ntts auth sms test <provider>` (sendet Test-OTP an Operator-Telefon), `ntts auth sms disable`.
  - **Role gates:** edit/test/disable = admin + MFA re-auth. View ohne secrets = admin + developer.
  - **Wir bauen keine eigene SMS-Treiber-Abstraktion** — GoTrue ist die Treiberschicht.
- **F-AUTH-12:** GoTrue fires `NOTIFY ntts_jwt_changed, '<user_id>'` on token refresh and on logout. Realtime container listens on this channel and evicts the Subscriber-Authorization-Snapshot-Cache entries for the affected user per [ADR-0026](./docs/adr/0026-realtime-three-layer-authorization-bucketing-snapshot-cache-bulk-eval.md). GoTrue-Fork-Patch — siehe [§3.1 Build vs Wrap vs Fork](#31-build-vs-wrap-vs-fork) und [ADR-0023 Addendum](./docs/adr/0023-monorepo-typescript-first-with-caddy-plugin-as-go-exception.md).

### 5.4 Storage (`/storage/v1`, `/storage/v3/s3`)

supabase-storage upstream.

- **F-STOR-1:** `POST /object/:bucket/:path` upload.
- **F-STOR-2:** `GET /object/:bucket/:path` (private) checks RLS.
- **F-STOR-3:** `GET /object/public/:bucket/:path` no auth.
- **F-STOR-4:** `POST /object/sign/...?expiresIn=3600` → signed URL.
- **F-STOR-5:** Bucket limits enforced before persist.
- **F-STOR-6:** S3-compatible API at `/storage/v3/s3` for third-party S3 SDKs.
- **F-STOR-7:** TUS endpoint for resumable uploads.
- **F-STOR-8:** Image transformations on signed URLs (`?width=…&height=…&format=…`).
- **F-STOR-9:** Per-bucket quotas: `storage.buckets.file_size_limit` (existing) plus `object_count_limit` (new). Enforced at upload time. Status in Studio Storage panel + `ntts storage buckets quota`.
- **F-STOR-11:** SHA-256 streaming-computed at upload time, persisted to `storage.objects.metadata->>'sha256'` (also exposed in object-list and `GET /object/info/...`). Hash computation uses the same read-buffer the persister consumes — no extra IO. **Kein automatisches Dedup**: derselbe Content unterschiedlicher Uploader bleibt zwei separate Objekte (separate Quota-Belastung, separate Delete-Semantik). Restore-Path `ntts backup restore --check-storage` re-hasht alle Objekte und vergleicht gegen Metadata — Mismatches → Exit-Code 2 + Liste in `_ntts.storage_integrity_failures`.
- **F-STOR-10:** Backup integration per [ADR-0022](./docs/adr/0022-storage-fs-snapshot-included-with-bounded-drift-and-s3-aware-backup.md):
  - **FS-backend:** `ntts backup snapshot` produces `pg_dump.tar.gz` + `caddy.tar.gz` + `storage.tar.gz` with bounded drift (rows created during the tar window may be metadata-only). Restore via `ntts backup restore`; optional `--check-storage` verify lists orphan metadata rows.
  - **S3-backend:** snapshot skips storage tar; Studio Backup page surfaces "S3 bucket backup is your responsibility (configured: …)".
  - **Override:** `NTTS_BACKUP_SKIP_STORAGE=true` for Operators using external backup tooling.
  - **Incremental snapshots:** v1.1+ roadmap.

### 5.4a Vault (`vault.*` schema)

Pgsodium-backed encrypted secrets for **SQL-tier consumers only** — PL/pgSQL, `wrappers` FDWs, pg_net callers, Studio-backend SQL. See [ADR-0008](./docs/adr/0008-three-classes-of-secrets.md).

- **F-VLT-1:** `vault.create_secret(value, name, description)` and `vault.update_secret(name, value)` available to `admin` Operator via Studio.
- **F-VLT-2:** Decryption only via `SECURITY DEFINER` wrapper functions; direct `SELECT * FROM vault.decrypted_secrets` requires `service_role`.
- **F-VLT-3:** Vault decrypt calls excluded from `pg_stat_statements` parameter capture.
- **F-VLT-4:** AI-provider key (ADR-0003) stored as `vault.secrets[ai_provider_key]`. Read by Studio-backend AI adapter only.
- **F-VLT-5:** Function code (TS in Edge Runtime) **does not** read Vault — use Function Secrets (F-FN-10) instead.

### 5.4c Wrappers / Foreign Data

Catalog-driven Studio UX, egress proxy enforcement, per-wrapper cost guardrails per [ADR-0016](./docs/adr/0016-wrappers-fdw-catalog-driven-with-egress-proxy-and-cost-guardrails.md).

- **F-FDW-1:** Wrappers (Stripe, S3, BigQuery, Firebase, etc.) and `postgres_fdw` configured via `/etc/ntts/wrappers.catalog.json` (versioned per NTTS release). Studio renders setup forms dynamically; CLI `ntts wrappers add/test/disable/delete/export/apply`.
- **F-FDW-2:** Configuration in `_ntts.wrappers_config (id, wrapper_id, server_name, server_options jsonb, user_mapping_vault_ref, foreign_tables jsonb, guardrails jsonb, enabled, ...)`. Credentials live in Vault; row stores a `vault.secrets[name]` reference. DDL synthesised by studio-backend.
- **F-FDW-3:** New container `ntts-egress-proxy` in compose stack. Postgres routes outbound HTTPS through proxy (env `https_proxy=…`). Allowlist in `_ntts.egress_allowlist (host_pattern, port, added_by, reason, wrapper_id)`. Wrapper enable auto-extends allowlist from catalog's `requires_egress_to`. Every connection logged to `_ntts.egress_log` (partitioned monthly).
- **F-FDW-4:** Per-wrapper cost guardrails in `wrappers_config.guardrails`: `statement_timeout_ms` (default 30000), `max_result_rows` (default 10000), `monthly_query_count_cap` (nullable). Enforced via `_ntts.wrapper_safe_select_<table>()` SECURITY DEFINER wrapping functions; direct foreign-table queries bypass guardrails by design (power-user opt-out).
- **F-FDW-5:** Role gates: edit/test/disable/delete = admin + MFA; reveal Vault credential = admin + MFA re-auth (audit row); view config without secrets = admin + developer; SELECT from foreign tables = default `service_role` only, admin extends GRANTs.
- **F-FDW-6:** Test: `ntts wrappers test <id>` runs catalog-declared smoke query, result in `_ntts.wrappers_test_log`. Soft-warn if `last_tested_at` older than 30 days.
- **F-FDW-7:** `ntts wrappers export > wrappers.toml` (credentials externalised as `${VAULT_REF:name}`), `ntts wrappers apply -f`.

### 5.4b Realtime (`/realtime/v1`)

Default-deny, RLS-derived for table channels, policy-engine for non-table channels per [ADR-0015](./docs/adr/0015-realtime-default-deny-rls-derived-table-channels-policy-engine.md).

- **F-RT-1:** Table-change broadcasts (`postgres_changes`) require explicit per-table opt-in via `_ntts.realtime_enabled_tables (schema, table, enabled_at, enabled_by)`. Pre-toggle UI check verifies RLS active + at least one policy.
- **F-RT-2:** Enabled tables: per row-change × per subscriber, Realtime sets the subscriber's JWT context and runs a primary-key `SELECT` against the changed row. Zero rows → skip broadcast. Non-zero → broadcast returned columns (RLS column-level filtering honoured via views). **Performance pipeline per [ADR-0026](./docs/adr/0026-realtime-three-layer-authorization-bucketing-snapshot-cache-bulk-eval.md):** Layer-1 topic-filter bucketing + Layer-2 Subscriber-Authorization-Snapshot-Cache + Layer-3 bulk-RLS-eval fallback collapse the per-row × per-subscriber fan-out to ~10–50 SQL-queries/s under mixed workload at the §7 capacity-target of 5.000 subscribers. Complex RLS-policies (subquery, `current_setting()`, time-dependent) fall through to Layer-3 with linear scaling — Advisor (F-RT-7) flags these as `realtime_uncached`.
- **F-RT-3:** Service-JWT subscribers bypass RLS as normal Postgres semantics dictate.
- **F-RT-4:** Non-table channels (`broadcast`, `presence`) governed by `_ntts.realtime_channel_policies (id, topic_pattern, action enum('subscribe','publish'), expression, description)`. Default-deny: no truthy matching ALLOW policy → DENY. Expression-Grammar formal definiert via [pgsql-ast-parser](https://github.com/oguimbal/pgsql-ast-parser)-basiertem Whitelist-Validator:
  - **Erlaubt:** Comparison-Operatoren (`= <> < <= > >= IN BETWEEN IS [NOT] NULL LIKE ILIKE ~`), Boolean (`AND OR NOT`), jsonb-Path-Access (`-> ->> #>>`), Type-Casts (`::text ::int ::uuid ::timestamptz ::boolean`), whitelisted Built-ins (`lower upper length coalesce array_position jsonb_path_exists starts_with split_part`), Variablen `request.jwt.claims` (jsonb) und `topic` (text).
  - **Blockiert:** Subqueries, FROM/JOIN, DML, DDL, Function-Defs, Dollar-Quotes, CTEs, `WITH RECURSIVE`, jede nicht-whitelisted Funktion (insbesondere `pg_sleep`, `current_setting`).
  - **Validation an zwei Punkten:** (1) Studio/CLI-Save: AST-Walker rejected mit erklärendem Fehler vor Insert. (2) Runtime: Realtime-Container ruft `_ntts.eval_realtime_policy(expression, jwt_claims, topic) returns boolean` SECURITY DEFINER, die WHERE-Klausel aufbaut und ausführt.
  - **Test-Tool** (F-RT-7) zeigt Grammar-Fehler inline beim Tippen.
- **F-RT-5:** Denies are sampled and rate-limited into `_ntts.realtime_policy_denies` for debugging; not audit-immutable; retention via F-LOG-1 pattern.
- **F-RT-6:** Realtime container trusts `X-NTTS-Auth` header from `ntts-edge` (ADR-0004); its own JWT-verification is disabled. Internal HMAC trust rotated as part of NTTS upgrade.
- **F-RT-7:** Studio "Realtime" UI exposes per-table toggle, channel-policy editor with PostgreSQL syntax highlighting, and a "Would this topic + JWT-claims be allowed?" test tool. **Cache-Telemetry per [ADR-0026](./docs/adr/0026-realtime-three-layer-authorization-bucketing-snapshot-cache-bulk-eval.md):** pro abonnierter Tabelle (a) Cache-Hit-Rate (L2-cached vs L3-fallback) der letzten 5 Minuten, (b) Active Subscriber Count pro `(channel, filter_hash)`-Bucket, (c) Top-10 slowest-evaluating Policies nach mittlerer L3-Eval-Zeit, (d) `realtime_uncached`-Marker für Policies die nicht als Snapshot-Expression darstellbar sind. CLI: `ntts realtime enable <table>`, `ntts realtime disable <table>`, `ntts realtime policy add/list/remove`, `ntts realtime cache-stats`.

### 5.5 Functions (`/functions/v1`)

- **F-FN-1:** `POST /functions/v1/:name` invokes a public Function. Private = function-to-function or event only.
- **F-FN-2:** Bundle stored in `_ntts.function_bundles (id, function_id, hash, eszip_bytes, created_at, created_by)`. Pointer in `_ntts.function_deployments.active_version_id`.
- **F-FN-3:** DB-event trigger → pg_net → internal HTTP call with service-JWT.
- **F-FN-4:** Cron trigger → pg_cron + pg_net → same path as F-FN-3.
- **F-FN-5:** Timeout default 30s, max 150s. Quota kill = V8 terminates user-worker. Parallel invocations untouched (own isolate).
- **F-FN-6:** Stdout/stderr → `_ntts.function_logs` (range-partitioned by `started_at`, daily) with `execution_id`, `function_id`, `version_id`, `started_at`, `duration_ms`, `cpu_ms`, `memory_peak_kb`, `exit_status`. Retention per [ADR-0013](./docs/adr/0013-logs-retention-two-class-partition-drop-drain-mirror.md).
- **F-FN-7:** Typed `db` client from live schema. User-JWT context (RLS) or service-JWT (event/cron). Types reflect schema shape, not RLS-filtered shape — see [ADR-0012](./docs/adr/0012-type-gen-flow-dedicated-worker-async-versioned.md).
- **F-FN-8:** Pre-deploy Pipeline (8 stages, hard-block): tsc → eslint → bundle → test → schema-compat → security-scan → **signature** → dry-run-deploy. Override needs `override_reason_category` + `override_reason_text` → `_ntts.function_audit` (siehe §10-Q8-Resolution) with `operator_id`, `failed_stage`.
  - **Signature-Stage:** Studio-backend signiert das Bundle mit Ed25519 (`NTTS_BUNDLE_SIGNING_KEY` aus Container-Env, *eigener Key, nicht Master*) über `(bundle_hash ‖ manifest_hash ‖ pipeline_run_id ‖ unix_timestamp)`. Persistiert nach `_ntts.function_bundles.signature_pipeline` (bytea NOT NULL). Pointer-UPDATE (F-FN-11) refused, wenn `signature_pipeline IS NULL` oder gegen aktuellen Public-Key invalide.
  - **Operator-Signatur (optional):** Wenn `_ntts.studio_users.signing_pubkey` gesetzt (Ed25519/age/ssh-ed25519), prompted Studio den Operator, die Pipeline-Run-Eingabe mit dem privaten Key zu signieren (lokal via Browser-WebCrypto oder CLI `ntts fn deploy --sign`); landet in `signature_operator` (bytea NULL ok).
  - **Verify-at-Boot:** Edge-Runtime-Hydratation (F-FN-14) prüft `signature_pipeline` gegen aktuellen Pipeline-Public-Key. Mismatch → Function-Status `signature_invalid`, andere Functions starten weiter (fail-closed nur für diese eine Function).
  - **Key-Rotation:** `ntts admin rotate-pipeline-signing-key` (MFA + reason) generiert neues Keypair; historische Bundles bekommen `signature_pipeline_legacy` Tag, Verify degradiert auf WARN-Alert (F-LOG-4) statt fail-closed bis Operator alle Functions neu deployt.
- **F-FN-9:** Sandbox quotas: CPU 5s/30s soft/hard, mem 256MB/1GB, disk 50MB tmpfs, egress allowlist, process-spawn block.
- **F-FN-10:** Secrets in `_ntts.function_secrets` (per-Function), pgcrypto-encrypted, master-key `NTTS_MASTER_KEY` in Edge Runtime container env only. See [ADR-0007](./docs/adr/0007-secret-injection-worker-boot-rotation-via-notify.md):
  - **Read timing:** at user-worker boot, not per-invocation. Worker env stable for worker lifetime.
  - **API in Function code:** `Deno.env.get('NAME')` primary; `process.env.NAME` via Deno compat shim.
  - **Never bundled:** secrets are not part of the eszip Bundle; manifest may reference secret *names*, never values.
  - **Rotation:** Operator changes a secret → `NOTIFY ntts_fn_secret_change` with `{function_id, name}` → Edge Runtime evicts affected workers → next invocation sees new value. Sub-second propagation in normal operation.
  - **Log redaction:** log pipeline replaces plaintext matches of secret values → `[REDACTED:NAME]`.
  - **Studio Reveal:** admin-only, requires MFA re-auth, lands `_ntts.studio_audit_read` row.
- **F-FN-11:** Atomic deploy = one transaction: INSERT Bundle + UPDATE pointer + `NOTIFY ntts_fn_deploy, '{function_id}'`. Keep last 20 Bundles per Function. Rollback = pointer UPDATE + NOTIFY. Audit row each time.
- **F-FN-12:** Schema-compat-check both directions, see [ADR-0005](./docs/adr/0005-bundle-manifest-hybrid-types-and-ast.md) and [ADR-0012](./docs/adr/0012-type-gen-flow-dedicated-worker-async-versioned.md):
  - **Primary check** = `tsc` against the live schema's generated `db`-types. Deploy-stage runs in the `ntts-typegen` worker; compares Bundle's `types_version` against current. If drifted, soft-recompiles against current schema. Green → deploy with `recompiled_from` audit field. Red → reject with diff.
  - **Secondary check** = Bundle manifest, stored in `_ntts.function_bundles.manifest jsonb`. Shape:
    ```json
    {
      "schema_hash": "sha256:...",
      "reads":  { "public.users":  ["id", "email"] },
      "writes": { "public.users":  ["banned_at"] },
      "rpcs":   ["public.search"],
      "extensions": ["pgvector"],
      "function_to_function": ["send_email"],
      "dynamic_sql": false
    }
    ```
  - **(a) Forward (Deploy):** Bundle manifest vs current schema → block if needed columns missing.
  - **(b) Reverse (Migration):** planned DDL vs union of active Bundles' manifests → block + list affected Functions + offer 1-click "sync all" (re-runs Pipeline against new schema in parallel).
  - **Dynamic SQL** in a Function → manifest carries `"dynamic_sql": true` → soft-warn, not hard-block. Override goes to `_ntts.function_audit`. ESLint rule `no-dynamic-table-name` makes the path deliberate.
  - **Function-to-Function** calls merge transitively at Bundle time. Cycles → hard-fail.
- **F-FN-13:** Main-worker holds `LISTEN ntts_fn_deploy`. On notify: SELECT active Version → fetch eszip bytes → write tmpfs → evict that Function's isolates. Next invocation recompiles.
- **F-FN-14:** Boot hydration: read all `_ntts.function_deployments WHERE active_version_id IS NOT NULL`, load Bundles to tmpfs, then start LISTEN. Single broken Function → `status: 'broken'`, others stay up.
- **F-FN-15:** `pg_dump` covers `_ntts.function_bundles`, `function_deployments`, `function_versions`, `function_audit`. No separate Function-backup volume.
- **F-FN-16:** Background tasks via `waitUntil` (Edge Runtime native). Quotas extend by 30s post-response.
- **F-FN-17:** OTel exporter configurable via env (`OTEL_EXPORTER_OTLP_ENDPOINT`).
- **F-FN-18:** Function-to-Function call, see [ADR-0006](./docs/adr/0006-function-to-function-internal-http-caller-identity-default.md):
  - SDK: `await ntts.functions.invoke(name, payload, { as?: 'caller' | 'service' })`. Default `as: 'caller'`.
  - Transport: internal HTTP to `http://edge-runtime:9000/_internal/invoke/:name`, HMAC-signed with key from Edge Runtime env. Endpoint not reachable through Gateway.
  - Identity propagation: caller's `X-NTTS-User-Id` + `X-NTTS-Role` headers forwarded by default; `as: 'service'` replaces them with service-role internals.
  - Recursion depth ≤ 5 via `X-NTTS-Call-Depth`; cycles caught at Bundle time (F-FN-12).
  - Auth-hooks (F-AUTH-7), DB-event triggers (F-FN-3), and cron triggers (F-FN-4) all use this endpoint with `as: 'service'`.
  - Failure modes are typed: `FunctionNotFound`, `FunctionTimeout`, `FunctionQuotaExceeded`, `FunctionRecursionLimit`.

### 5.6 Studio User Management

- **F-STU-1:** `_ntts.studio_users (id uuid PK, email unique, password_hash, role enum('admin','developer','viewer'), mfa_secret nullable, created_at, last_login_at, disabled_at)`.
- **F-STU-2:** **Setup Token bootstrap.** On first `up` with empty `studio_users`, control-plane writes a single-use Setup Token to its stdout. `/setup?token=…` lets the first human create the initial admin Operator. Token expires 24h or on use. **Non-interactive provisioning:** if `NTTS_SETUP_TOKEN_FILE=/path/to/file` is set in the studio-backend container's env, the token is *also* written to that path with mode `0600` (container user). Post-consume / post-expire, the file is overwritten in-place with the literal string `CONSUMED` or `EXPIRED` (not unlinked, so Ansible/Terraform can poll state).
- **F-STU-3:** MFA TOTP required for `admin`, optional for `developer`/`viewer`.
- **F-STU-4:** `admin` Operator can invite further Operators by email. Invitation = signed link, expires 24h. SMTP-required for email-invite path; if SMTP unconfigured, admin can generate single-use claim-links from studio UI as fallback.
- **F-STU-5:** Disable, don't delete — `disabled_at` set, all studio-session JWTs for that user revoked. Preserves audit referential integrity.
- **F-STU-6:** Role gates:
  - `admin`: everything incl. Operator-management, Function secrets, Pre-deploy Pipeline override, Vault read.
  - `developer`: schema, Functions, deployments, Pipeline override with reason.
  - `viewer`: read-only (schema, logs, monitoring); UI hides all write affordances.
- **F-STU-7:** Every PII-read by an Operator (open `auth.users` list, read a log row containing App-User data) writes to `_ntts.studio_audit_read (id, prev_hash, payload, payload_hash)`. `payload` carries `(studio_user_id, action, target_kind, target_id, accessed_at)` — the **fact of access**, not the accessed content.
- **F-STU-8:** Every write action by an Operator writes to `_ntts.studio_audit_write` with the same hash-chained shape plus `diff` or sanitised `payload` (PII redacted).
- **F-STU-10:** Audit tables are append-only — see [ADR-0009](./docs/adr/0009-audit-immutability-local-hash-chain-optional-drain.md):
  - `GRANT INSERT, SELECT` only on audit tables to all application roles.
  - `BEFORE UPDATE OR DELETE` trigger raises exception.
  - Hash-chain (`prev_hash` + `payload_hash`) on each row; INSERTs serialised via `pg_advisory_lock`.
  - `_ntts.verify_audit_chain(table_name)` SQL function; CLI `ntts audit verify`; Studio "Audit Health" panel.
  - Optional external drain via `NTTS_AUDIT_DRAIN=s3://...` or `syslog://...` for Operators who need tamper-prevention rather than tamper-evidence.
  - **Schema-mutating NTTS-upgrades write a chain-reset marker.** When an internal migration alters the column set or canonical hash-input of any `_ntts.studio_audit_*` or `_ntts.function_audit` table, the migration writes an `_ntts.audit_chain_resets (table_name, reset_at, reason, ntts_version_from, ntts_version_to)` row in the same transaction (`reason='ntts_upgrade'`). `verify_audit_chain` reads chains segment-wise across these markers (consistent with the `ntts audit truncate` path in F-LOG-3). See [ADR-0011](./docs/adr/0011-upgrades-via-ntts-upgrade-two-migration-streams.md) audit-chain-reset clause.
- **F-STU-9:** `auth.users` and `_ntts.studio_users` are independent. An App User does not have studio access. An Operator does not have an App User row.

### 5.7 GDPR Hard-Delete (`/admin/gdpr/delete-user`)

- **F-GDPR-1:** Endpoint accepts an App User id. Wipes `auth.users`, foreign-key-cascade through application tables (operator declares the cascade map in `_ntts.gdpr_cascade`).
- **F-GDPR-2:** Function logs containing the user's id are tombstoned: row kept, payload redacted.
- **F-GDPR-3:** Audit logs (`studio_audit_write`, `studio_audit_read`) are **not** wiped — they record Operator actions, not App User data. Operator action on App User stays auditable.
- **F-GDPR-4:** PITR backups older than the retention window (default 30 days) are *not* purged on delete; operator runs the redaction script on backup restore.

### 5.8 Migration & Schema Management

Forward-only, source-of-truth in the application repo. See [ADR-0010](./docs/adr/0010-migrations-forward-only-files-in-repo.md).

- **F-MIG-1:** Files at `migrations/NNNN_slug.sql`. Optional companion `migrations/NNNN_slug.test.sql` runs in Pipeline test stage.
- **F-MIG-2:** `_ntts.schema_migrations (version int PK, name, hash sha256, applied_at, applied_by)` tracks applied state.
- **F-MIG-3:** Each migration runs in a transaction by default. Header pragma `-- ntts:no-transaction` opts out (e.g. `CREATE INDEX CONCURRENTLY`). No-tx migrations cannot rollback on failure — explicit warning in CLI.
- **F-MIG-4:** F-FN-12 schema-compat-check runs **inside** the migration transaction. Compat failure → automatic rollback + report of blocked Functions.
- **F-MIG-5:** Successful COMMIT triggers post-migration chain (async; see [ADR-0012](./docs/adr/0012-type-gen-flow-dedicated-worker-async-versioned.md)):
  - `NOTIFY pgrst, 'reload schema'` (PostgREST + pg_graphql refresh)
  - `NOTIFY ntts_typegen` → `ntts-typegen` worker regenerates `_ntts.generated_types` (rows `name='db'` and `name='functions'`), bumps `types_version`, writes new `types_fingerprint`. Migration COMMIT does NOT wait.
  - `NOTIFY ntts_policy_changed, '<table_oid>'` for every table whose RLS policy set was added/altered/dropped by the migration. Realtime container listens and evicts Subscriber-Authorization-Snapshot-Cache entries for that table per [ADR-0026](./docs/adr/0026-realtime-three-layer-authorization-bucketing-snapshot-cache-bulk-eval.md).
  - `pg_event_trigger` Backstop catches raw DDL outside the migration path; same regeneration flow (incl. `ntts_policy_changed` fan-out), increments `_ntts.out_of_band_ddl_count`.
  - `ntts-typegen` publishes WS event `types:changed` → Studio Editor pulls fresh types on next save/debounce.
- **F-MIG-6:** No down-migrations. A wrong migration is corrected by a new forward migration.
- **F-MIG-7:** File-hash drift detection: if applied migration's file content diverges from `schema_migrations.hash`, CLI warns at next run.
- **F-MIG-8:** `ntts db status` compares structural hash of current schema vs latest applied migration; reports manual drift.
- **F-MIG-9:** Role gates:
  - `admin` + `developer`: can apply migrations.
  - `viewer`: can read migration list; Apply buttons disabled.
- **F-MIG-10:** 1-click "sync affected Functions": opens listed Functions in Studio editor + queues re-deploy. No automated code rewriting.

### 5.8c Container Healthchecks & Aggregated Status

Per-container Docker healthchecks + startup ordering + aggregated GREEN/YELLOW/RED status per [ADR-0019](./docs/adr/0019-container-healthchecks-startup-ordering-aggregated-status.md).

- **F-HC-1:** Every compose service carries `healthcheck:` (10s interval, 3s timeout, 3 retries, 30s start_period). Service-specific tests defined in ADR-0019.
- **F-HC-2:** Startup graph via `depends_on: condition: service_healthy`: postgres → supavisor → (postgrest, gotrue, storage, realtime, edge-runtime, ntts-typegen) → studio-backend → ntts-edge.
- **F-HC-3:** NTTS-internal watcher in studio-backend polls `docker ps` every 30s, writes `_ntts.container_status (name, state, health, last_started_at, restart_count_10m, updated_at)`. Requires `/var/run/docker.sock` mounted read-only into studio-backend.
- **F-HC-4:** 3 consecutive unhealthy → auto-restart + ERROR alert. 3 restarts/10min → CRITICAL, no further auto-restart, Operator action required. Postgres exempt: unhealthy raises ERROR immediately, restart requires admin + MFA confirmation.
- **F-HC-5:** Aggregated status via `_ntts.aggregate_health()` SQL function: GREEN (all healthy, no ERROR/CRITICAL alerts, pg_cron alive, TLS >7d, PITR <30min lag), YELLOW (≥1 WARN, else GREEN), RED (≥1 ERROR/CRITICAL or container unhealthy or pg_cron stale).
- **F-HC-6:** Surfaces: `ntts status` CLI, Studio "Health" page (with admin Restart/Force-healthcheck actions), Gateway `/healthz` returning JSON with status; `/healthz?strict=true` returns 503 for YELLOW or RED for external load-balancers needing fail-closed degraded handling. Prometheus `/metrics` exposes `ntts_container_healthy` + `ntts_alert_open` gauges.

### 5.8b Postgres Roles & Superuser Access

Three-tier access model per [ADR-0018](./docs/adr/0018-postgres-superuser-three-tier-access-with-nss-admin-default.md).

- **F-DB-1:** `postgres` SUPERUSER bound to Unix-socket-only inside the Postgres container; password random-generated at first `up`, stored in `_ntts.platform_secrets[postgres_superuser]` (pgcrypto with `NTTS_MASTER_KEY`). Override via `NTTS_BOOTSTRAP_POSTGRES_PASSWORD` env at first boot; cleared from process env after migration to platform_secrets.
- **F-DB-2:** `nss_admin` role: `NOSUPERUSER`, `BYPASSRLS`, broad grants on `_ntts.*`, `public.*`, service schemas; explicitly denied `REPLICATION`, `ALTER USER postgres`, superuser-creating `CREATEROLE`. Operator routine SQL runs as this role.
- **F-DB-3:** Standard Supabase service-owner roles (`supabase_auth_admin`, `supabase_storage_admin`, `supabase_realtime_admin`, `supabase_functions_admin`) and app-tier roles (`anon`, `authenticated`, `service_role`) created at first `up` per upstream template.
- **F-DB-4:** Three access tiers: (1) Studio SQL Editor as `nss_admin` with audit; (2) `ntts admin shell` / `ntts admin sql` as `nss_admin` through Gateway with audit; (3) `ntts admin shell --superuser` requires MFA re-auth + non-empty reason + 30s cool-down, connects via `docker exec` over Unix socket, every statement audit-logged with reason + session-id, banner notification fanned out to all open admin Studio sessions via `NOTIFY ntts_admin_event`.
- **F-DB-5:** `ntts admin rotate-superuser-password` (MFA + reason + cool-down) rotates `postgres` password in platform_secrets.
- **F-DB-6:** Pre-boot health-check refuses startup if `platform_secrets[postgres_superuser]` is unreadable (master-key mismatch or row missing).
- **F-DB-7:** Default-GRANT-Matrix (kanonisch in `0001_init.sql` + `0002_role_grants.sql`; Migrations-Linter aus ADR-0024 enforced sie):

  | Role | `_ntts.*` | `public.*` | `auth.*` | `storage.*` | `realtime.*` | `vault.*` |
  |---|---|---|---|---|---|---|
  | `anon` | NONE | NONE (RLS-on-create) | NONE | NONE | NONE | NONE |
  | `authenticated` | NONE | NONE (RLS-on-create) | NONE | NONE | NONE | NONE |
  | `service_role` | NONE | ALL (RLS-bypass) | NONE | ALL via storage-api | ALL via realtime-api | NONE |
  | `nss_admin` (F-DB-2) | ALL (BYPASSRLS) | ALL | LIMITED (audit-mediated) | LIMITED | LIMITED | LIMITED |
  | `supabase_auth_admin` | NONE | NONE | ALL (owner) | NONE | NONE | NONE |
  | `supabase_storage_admin` | NONE | NONE | NONE | ALL (owner) | NONE | NONE |
  | `supabase_realtime_admin` | NONE | NONE | NONE | NONE | ALL (owner) | NONE |
  | `supabase_functions_admin` | NONE | NONE | NONE | NONE | NONE | NONE |
  | `postgres` (SUPERUSER) | ALL | ALL | ALL | ALL | ALL | ALL |

  Studio-/CLI-Migrations-Linter (ADR-0024) refused jede `GRANT … TO anon|authenticated` auf `_ntts.*`, `auth.*`, `storage.*`, `realtime.*`, `vault.*`.

### 5.8a Logs-Retention & Disk-Health

Two-class policy per [ADR-0013](./docs/adr/0013-logs-retention-two-class-partition-drop-drain-mirror.md).

- **F-LOG-1:** `_ntts.function_logs` retention is hybrid: time-based (`NTTS_FUNCTION_LOGS_RETENTION_DAYS`, default 30) AND size-cap (`NTTS_FUNCTION_LOGS_SIZE_CAP_GB`, default 10). `ntts_log_pruner()` pg_cron job runs hourly; drops oldest day-partitions when either bound trips.
- **F-LOG-2:** Audit tables (`studio_audit_read`, `studio_audit_write`, `function_audit`) are range-partitioned monthly. Never auto-deleted. Optional continuous mirror to `NTTS_AUDIT_DRAIN=s3://… | syslog://…`; each sink-write writes `_ntts.audit_drain_receipts` row.
- **F-LOG-3:** `ntts audit truncate --before YYYY-MM-DD` is a separate, MFA-gated, partition-aware command. Refuses unless all rows older than cutoff have receipts. Writes `_ntts.audit_chain_resets` row so `verify_audit_chain` reads the chain in segments.
- **F-LOG-4:** Alert engine: pg_cron `_ntts.check_health()` every 5 min computes disk-used pct, function_logs size, drain lag, PITR lag, partition counts. Inserts/clears rows in `_ntts.system_alerts (severity, kind, payload, raised_at, cleared_at)`. NOTIFY `ntts_health` on state change. Default thresholds: disk WARN 70% / ERROR 85% / CRITICAL 95%; drain-lag WARN 60min / ERROR 1440min; PITR-lag WARN 30min.
- **F-LOG-5:** Severity actions. WARN → Studio header yellow badge. ERROR → red badge + email (if SMTP) + webhook (if configured). CRITICAL → ERROR actions + automatic function_logs aggressive-prune. Traffic is never blocked by disk pressure.
- **F-LOG-6:** Consumers: Studio header polls `system_alerts`; CLI `ntts health` reads same table; Prometheus endpoint `/metrics` on studio-backend exposes table sizes, drain lag, partition counts, alert counts by severity.
- **F-LOG-7:** Configurability live-overridable via Studio Admin UI and `ntts logs retention set`/`ntts health threshold set`; persisted in `_ntts.config`.

### 5.9 CLI `ntts`

The CLI is the primary operator interface for headless, CI, recovery, and scripted workflows. **Coverage guarantee: every state-mutating Studio action has a CLI equivalent.** UI-only is acceptable for visual surfaces (AI assistant, ERD designer); state-change-only-in-UI is not.

- **F-CLI-1:** Auth via `ntts login [--url ...]` → browser flow against Studio login (password + MFA) → refresh-token written to `~/.config/ntts/credentials.toml` (`chmod 600`). Service-mode via `NTTS_SERVICE_KEY` env + `--service`.
- **F-CLI-2:** Config hierarchy:
  - `~/.config/ntts/credentials.toml` (per-Operator: URLs, tokens)
  - `./ntts.config.toml` (per-app-repo: migration dir, types output, pipeline config)
  - `./functions/<name>/ntts.fn.toml` (per-Function: secrets-needed, quota overrides, schedule)
- **F-CLI-3:** Role gates enforced server-side at the Gateway; CLI surfaces the result clearly.
- **F-CLI-4:** Output `--output json|yaml|table`; default `table`. Exit codes: 0 success, 1 user-error, 2 server-error, 3 auth-error.
- **F-CLI-5:** CI env vars: `NTTS_URL`, `NTTS_SERVICE_KEY`, `NTTS_OPERATOR_TOKEN`.
- **F-CLI-6:** Command surface (each command is a state-mutation primitive or read-only query):

  | Group | Commands |
  |---|---|
  | Lifecycle | `up`, `down`, `status`, `logs <service>` |
  | Auth | `login`, `logout`, `whoami` |
  | Init | `init` (scaffold app repo), `dev` (local hot-reload), `docs gen` |
  | DB | `db migrate`, `db status`, `db make <slug>`, `db verify` |
  | Types | `types generate`, `types watch` |
  | Functions | `fn deploy`, `fn list`, `fn rollback`, `fn invoke`, `fn logs`, `fn scan` |
  | Function Secrets | `fn secrets set`, `fn secrets ls`, `fn secrets rotate`, `fn secrets reveal` (MFA re-auth) |
  | App Users | `users invite`, `users ls`, `users delete` (= GDPR endpoint) |
  | Operators | `operators invite`, `operators ls`, `operators disable` (admin only) |
  | Storage | `storage buckets`, `storage upload`, `storage download` |
  | Vault | `vault set`, `vault ls`, `vault rotate` (admin only) |
  | Audit | `audit verify`, `audit tail`, `audit truncate --before YYYY-MM-DD` (MFA) |
  | Auth OAuth | `auth oauth list`, `auth oauth set <provider>`, `auth oauth test <provider>`, `auth oauth disable <provider>`, `auth oauth delete <provider>` (MFA), `auth oauth export`, `auth oauth apply -f` |
  | Wrappers | `wrappers list`, `wrappers add <id>`, `wrappers test <id>`, `wrappers disable <id>`, `wrappers delete <id>` (MFA), `wrappers export`, `wrappers apply -f`, `wrappers egress list/add/remove` |
  | Logs | `logs retention set --days N --cap MGB`, `logs retention show` |
  | Health | `health`, `health threshold set <kind> <value>` |
  | TLS | `tls list`, `tls set <domain>`, `tls challenge set <domain>`, `tls renew <domain>` |
  | Admin SQL | `admin sql --file <f>`, `admin shell`, `admin shell --superuser` (MFA + reason + cool-down), `admin rotate-superuser-password` |
  | Backup | `backup snapshot`, `backup restore`, `backup pitr-status` |
  | Upgrade | `upgrade [--to vX.Y]` (NTTS version bump), `upgrade-self` (CLI binary) |

- **F-CLI-8:** Distribution. Primary: Single-binary via `bun --compile` for `linux-x64`, `linux-arm64`, `darwin-x64`, `darwin-arm64`, `windows-x64`; installed via `curl -fsSL https://get.ntts.dev | sh` (downloads platform binary into `/usr/local/bin/ntts`, `chmod +x`, prints version). Secondary: `@ntts/cli` on npm for Node-native CI use (`npx @ntts/cli ...`). Both built from one TS codebase under `apps/cli/`. `ntts upgrade-self` checks GitHub Releases for newer binaries and replaces in-place (atomic rename, falls back to print-instructions on Windows).
- **F-CLI-7:** `ntts fn scan` outputs:
  - License-check against project allowlist (default MIT/Apache/BSD/PostgreSQL/ISC; configurable in `ntts.config.toml`).
  - Bundle-size: warn > 10 MB, error > 50 MB.
  - Static-secrets scan: regex-based hits on common API-key patterns committed in source → fail with line numbers.
  - SBOM export (`--out sbom.json`, CycloneDX or SPDX).

### 5.10a NTTS Upgrades

See [ADR-0011](./docs/adr/0011-upgrades-via-ntts-upgrade-two-migration-streams.md).

- **F-UPG-1:** Two migration streams. App migrations in `migrations/` (Operator-owned, tracked in `_ntts.schema_migrations`). NTTS-internal migrations in `_ntts/migrations/` embedded in the image (tracked in `_ntts.internal_migrations`). **Kanonische `_ntts.*`-DDL ist `apps/studio-backend/src/internal-migrations/0001_init.sql`** — das ist die Source-of-Truth für alle `_ntts.*`-Tabellen, Functions, Trigger; PRD-Features referenzieren Tabellen + Spalten per Name, die SQL-Datei ist autoritativ. Schema-ERD als Begleit-Doku in `docs/schema/erd.md`.
- **F-UPG-2:** `_ntts.version (current_version, applied_at, applied_by)` marker. Each NTTS release declares a compatibility range `MIN_KNOWN..MAX_KNOWN`. Boot refuses to start if marker is out-of-range.
- **F-UPG-3:** `ntts upgrade [--to vX.Y]` orchestrates: pre-check → auto-snapshot → image pull → upgrade-prep → upgrade-apply (in TX) → verify.
- **F-UPG-4:** Failure before COMMIT → automatic compose-revert. Failure post-COMMIT → `ntts upgrade --rollback` restores from snapshot.
- **F-UPG-5:** `ntts upgrade --check` parses bundled `RELEASE_NOTES.md`, lists breaking changes, requires `--confirm-breaking` flag for breaking upgrades.
- **F-UPG-6:** Postgres-major upgrade is a separate path (`ntts upgrade --postgres-major`) with explicit Operator confirmation and longer accepted downtime.
- **F-UPG-7:** Skip-version paths (e.g. v1.0 → v1.3 in one jump) are CI-tested; documented when a path requires a stepping-stone version.
- **F-UPG-8:** During upgrade, all external traffic returns 503 with `Retry-After`. Typical duration 1–2 min. Acknowledged appliance downtime.

### 5.10 SDK

- **F-SDK-1:** `createClient(url, anonKey)` matches `@supabase/supabase-js` shape.
- **F-SDK-2:** TS generics from schema file.
- **F-SDK-3:** Thin re-export of `@supabase/supabase-js` + NTTS type augmentation.
- **F-SDK-4:** Mobile SDKs re-export upstream Supabase SDKs and surface NTTS-generated types per platform (Swift Codable / Kotlin data classes / Dart classes / Python TypedDicts).
- **F-SDK-5:** Distribution & Versioning. JS-SDK: npm `@ntts/client`. Mobile-SDKs in Standard-Registry pro Plattform: Swift via SPM (`ntts-swift` GitHub-Tag), Kotlin/Android via Maven Central (`dev.ntts:ntts-kotlin`), Flutter/Dart via pub.dev (`ntts`), Python via PyPI (`ntts-py`). **Versionierung lockstep mit NTTS-Minor** (1.3.x↔1.3.x cross-package); Patch-Versionen unabhängig für SDK-only-Bugfixes. CI: jeder NTTS-Minor-Release triggert SDK-Rebuild + Publish auf alle Registries (Trusted-Publisher OIDC für PyPI, Sonatype für Maven, signed Tag für SPM, `dart pub publish` für pub). Code-Layout: `mobile-sdks/{swift,kotlin,dart,python}/` als Sub-Trees im Monorepo, eigene Toolchain pro Platform.

---

## 6. Acceptance

1. `@supabase/supabase-js` + `createClient(NTTS_URL, NTTS_ANON_KEY)` → auth signup, login, REST select, storage upload, work, no code change.
2. `pg_dump --schema-only` from Supabase Cloud imports clean. RLS + triggers intact.
3. Phone-OTP user without email → `GET /auth/v1/user` returns `email: null`, `phone: "+49…"`.
4. pg_cron triggers Function → row appears in `_ntts.function_logs` with correct `execution_id`.
5. TS errors in editor → squiggles in ≤500ms, save blocked in strict mode.
6. Failed test → Pipeline blocks. Override → audit row with reason.
7. `while(true){}` → user-worker killed in ≤30s. Parallel invocations unaffected.
8. `console.log(SECRET)` → log shows `[REDACTED:SECRET_NAME]`.
9. Migration drops a column used by ≥1 deployed Function → migration blocked, affected Functions listed.
10. Rollback v17 → v16 → all invocations on v16 within 30s. Audit log has reason.
11. Deploy → cache invalidation propagates in <500ms (NOTIFY + bundle fetch).
12. `pg_dump` → restore in fresh Postgres → backend wakes up with all Functions, auth configs, buckets, Operators exactly as before.
13. App using `.channel().on('postgres_changes', …).subscribe()` works unchanged.
14. App using `.rpc('fn_name', { … })` works unchanged.
15. App using GraphQL (`urql`/`graphql-request`) against `/graphql/v1` returns RLS-respecting results.
16. App using Supavisor connection string at `:6543` connects and pools.
17. Fresh `docker compose up` → Setup Token printed to stdout → `/setup?token=…` flow → first admin Operator created → MFA TOTP enrolled.
18. Admin invites a developer Operator; developer logs in; developer overrides Pipeline with reason; `_ntts.function_audit` row carries developer's `studio_user_id`.
19. Viewer Operator can read schema and logs; UI hides write buttons; API returns 403 on write attempts.
20. Operator opens `auth.users` list → `_ntts.studio_audit_read` gets a row.
21. `NTTS_AI_PROVIDER` unset → AI features absent in studio. Set to `ollama` with endpoint → SQL editor shows assistant.
22. PostGIS `SELECT ST_AsText(geom)` works on `ntts-postgres:full`, returns clear "extension not installed" on `:slim`.
23. PITR restore to 30 min ago succeeds; `pg_dump` restore also succeeds; both produce identical result.
24. `/admin/gdpr/delete-user` for a given id → `auth.users` row gone, declared cascade tables redacted, `function_logs` tombstoned, audit rows preserved.
25. Mobile SDK quickstart (Swift in v1.0) builds, runs, signs a user in.
26. 1.000 Realtime-Subscribers auf einer Tabelle mit RLS `user_id = auth.uid()`, 50 INSERTs/s durch Users X1..X50: jeder Subscriber sieht nur eigene Rows, Postgres-Load <500 RLS-Queries/s gemessen via `pg_stat_statements` — validiert die ADR-0026 Three-Layer-Authorization-Pipeline gegen §7 Capacity-Target.

---

## 7. Non-Functional

| Category | Requirement |
|---|---|
| **Performance** | API P95 <100ms at <100 RPS on 4 vCPU / 8 GB VPS (single-node). Capacity-Dichte-Targets (single-node, MVP, Operator-Erwartung — nicht hard SLA): Concurrent App-User-Sessions 10.000 / Deployed Functions 500 / Realtime concurrent Subscribers 5.000 / Tabellen in `public.*` 1.000 / Storage FS-Objects 1.000.000 / API RPS sustained 100, Burst (<60s) 500 / Function-Invocations 50 RPS / aktive pg_cron-Jobs 100. CI Smoke-Tests an Sub-Targets, voller Scale-Test als separater Workflow |
| **Availability** | Single-node default 99.5%. Cluster Mode (replicas + multi-node Realtime) optional, operator-managed. RPO/RTO-Matrix pro Backup-Konfig: PITR aktiv → RPO 1 min, RTO 30 min; `pg_dump` nightly only → RPO 24 h (worst case), RTO 30 min; Cluster Mode (B6, Hot-Standby) → RPO <5 sec, RTO <60 sec (Promote). Setup-Wizard zeigt für gewählte Konfig die resultierenden Targets. Acceptance-Test §6 #23 erweitert: PITR-Restore verliert nicht mehr als 60 sec Daten in standardisiertem Last-Test |
| **Security** | App-User tokens HttpOnly+Secure. Operator passwords argon2id (memory_cost ≥64MB). MFA TOTP required for admin |
| **GDPR** | Per-user hard-delete endpoint (F-GDPR-1..4). Audit log append-only + hash-chained; Postgres superuser is the trust root, external drain optional — see [ADR-0009](./docs/adr/0009-audit-immutability-local-hash-chain-optional-drain.md) |
| **Observability** | JSON logs, request-IDs, `pg_stat_statements`, Prometheus, OTel |
| **Portability** | Reproducible migrations + seed. `docker compose up` <60s |
| **Compat** | Smoke-test suite with `@supabase/supabase-js` + mobile SDKs as CI gate |
| **License** | Apache 2.0 for NTTS-distributed code. Postgres extensions exempt — see [ADR-0001](./docs/adr/0001-license-policy-extensions-exempt.md) |
| **Upstream stability** | Pin all wrapped versions. CI smoke-test on each upstream major |

---

## 8. Risks

| Risk | Prob | Impact | Mitigation |
|---|---|---|---|
| SDK breaks on Supabase major update | Med | High | CI gate, pin versions |
| Upstream breaking changes (PostgREST/GoTrue/Realtime/Storage) | Med | Med | Pin versions, smoke-tests, gateway adapter |
| Logical-replication slot fills → DB stall | Med | High | Monitor `confirmed_flush_lsn`, alerts |
| RLS gap → data leak | Med | High | Multi-user test suite, fail-closed default |
| Function escapes V8 isolate | Low | High | Edge Runtime sandbox + egress allowlist |
| Storage FS fills up | Med | Med | Per-bucket quota, disk metric, 80% alert |
| Auth JWT secret compromised | Low | High | Rotation runbook, invalidate all sessions |
| Bundle bytea in Postgres too large | Low | Med | S3 fallback, 10 MB warn |
| Edge Runtime beta breaks | High | Med | Pin tag version, own test suite |
| Operator password DB compromise | Med | High | argon2id, MFA required for admin |
| Setup Token leaked from logs | Low | High | Single-use, 24h expiry, redacted after consumption |
| AI provider key leak | Low | High | Stored in `_ntts.studio_secrets`, pgcrypto-encrypted, never logged |
| WAL-G S3 destination misconfigured | Med | High | Pre-flight check at PITR-enable, alert on backup-lag |
| PostGIS license confusion downstream | Low | Med | `:full` image clearly labelled; slim image is the MIT-pure marketing path |

---

## 9. Out of Scope (architectural, not "later")

| Feature | Why |
|---|---|
| Multi-tenancy / workspaces | Single-Deployment by definition |
| Billing / usage tracking | Self-hosted, no pricing |
| OAuth server mode (NTTS as IdP) | We consume OAuth, don't provide it |
| SOC 2 / HIPAA programs | Operator concern, not feature |
| Branching ("projects" inside one box) | Single Deployment, no multi-env |
| Terraform provider | Appliance, not cloud resource |
| NTTS-hosted AI inference | EU sovereignty — Operator brings their own provider, see [ADR-0003](./docs/adr/0003-ai-assistant-provider-agnostic-default-off.md) |

Multi-env (dev/staging/prod) = Operator runs multiple Deployments. Not an NTTS feature.

---

## 10. Open Questions

1. Default anon privileges — **resolved** per [ADR-0024](./docs/adr/0024-deny-by-default-anon-and-rls-on-create.md): `_ntts.*` hard-locked from `anon`/`authenticated`; `public.*` tables RLS-on-create with no policies; `ntts db verify` soft-warns missing RLS.
2. Backup encryption — **resolved**: Default-on. Separater `NTTS_BACKUP_KEY` (nicht `NTTS_MASTER_KEY`). WAL-G via libsodium, `pg_dump` durch [age](https://age-encryption.org) gepiped, beide Sub-Keys via HKDF aus `NTTS_BACKUP_KEY` deriviert. Opt-out via `NTTS_BACKUP_ENCRYPTION=false` für Operators mit bereits-verschlüsseltem Backup-Storage. Boot-Check: wenn Encryption an, Key fehlt → WARN-Alert (F-LOG-4) + rotes Banner in Studio Backup-Page. Key-Escrow-Empfehlung in Operator-Doku.
3. Storage object hashing — **resolved**: SHA-256 wird bei jedem Upload streaming-berechnet, in `storage.objects.metadata->>'sha256'` persistiert. **Kein automatisches Dedup** im MVP (separate Quota- und Delete-Semantik wichtiger als Storage-Ersparnis). Backup-Restore-Check (`ntts backup restore --check-storage`) nutzt Hash für Integrity-Verify.
4. SMS provider abstraction — **resolved** in F-AUTH-11: catalog-driven Studio-UI über GoTrue's native SMS-Treiber (Twilio, Twilio Verify, MessageBird, Vonage, TextLocal). Keine eigene Treiber-Abstraktion.
5. Setup Token delivery — **resolved** in F-STU-2: stdout default; opt-in additional file write via `NTTS_SETUP_TOKEN_FILE` env, post-consume overwritten to `CONSUMED`/`EXPIRED`.
6. Cluster Mode write-path — **resolved** per [ADR-0025](./docs/adr/0025-cluster-mode-supavisor-as-readwrite-splitter.md): Supavisor ist der Read/Write-Splitter; PostgREST und Direct-Clients gehen alle durch den Pooler; HTTP-Layer bleibt unwissend.
7. SDK npm name — **resolved**: scoped `@ntts/*` family: `@ntts/client` (JS-SDK), `@ntts/cli`, `@ntts/types`. Konsistent mit Supabase-Konvention.
8. Override reason — **resolved**: Enum + Pflicht-Free-Text. Enum-Werte `bug-hotfix | test-flake | external-dependency | urgent-business | infrastructure-issue | migration-required | other`. Free-Text 20–1000 Zeichen Pflicht. Studio Submit-Button disabled bis beide gesetzt. `_ntts.function_audit.override_reason_category` (enum) + `override_reason_text` (text).
9. AI assistant prompt boundary — **resolved** per [ADR-0020](./docs/adr/0020-ai-assistant-layered-prompt-composition-per-tool-with-provider-adapters.md): layered (hardcoded base + Operator-editable per-tool templates in `_ntts.ai_assistant_tools`), provider-adapters for OpenAI/Anthropic/Mistral/Ollama, full audit in `_ntts.ai_assistant_log`.
10. SAML — **resolved** in F-AUTH-9: multi-IdP mit Email-Domain-Affinity-Routing über GoTrue.

---

## 11. Roadmap

Engineering convenience only — no scope phasing. All items in §4 are MVP. `B0..B6` are internal milestones to the **first public v1.0 release**. From v1.0 onwards, Operator-facing versions follow semver (see [ADR-0011](./docs/adr/0011-upgrades-via-ntts-upgrade-two-migration-streams.md)).

| Stage | Content |
|---|---|
| **B0** | Repo, compose (Postgres + PostgREST + pg_graphql + Supavisor + GoTrue + storage + caddy + realtime), `_ntts` schema, Setup Token bootstrap, health |
| **B1** | Control-plane, studio-backend, studio-frontend skeleton with login + MFA + 3 Roles, migrations, type-gen, CLI `ntts` |
| **B2** | Edge Runtime wrap, Bundles, NOTIFY invalidation, Pre-deploy Pipeline, versioning + rollback, Function secrets, schema-compat, editor with live TS |
| **B3** | Realtime wrap (channels, postgres changes, presence, broadcast), Storage S3/TUS/image-transform, all GoTrue OAuth providers + SAML, custom SMTP + email templates |
| **B4** | AI-assistant adapter, Advisor, log explorer, auto API docs, Backups UI, GDPR hard-delete endpoint, RLS sandbox, ERD |
| **B5** | TLS + custom domains (Caddy), `pg_dump` + WAL-G PITR runbook, Prometheus, OTel, mobile SDKs |
| **B6** | Cluster Mode (Read Replicas, multi-node Realtime), `:full` image (PostGIS, pgroonga, plv8, plpython), Bundle storage in S3 |

---

## Sources

- supabase.com/features
- supabase.com/docs/guides/self-hosting
- supabase.com/docs/guides/auth
- supabase.com/docs/guides/database/extensions
- supabase.com/docs/guides/realtime
- supabase.com/docs/guides/storage
- github.com/supabase/edge-runtime
- github.com/supabase/auth
- github.com/supabase/storage
- github.com/supabase/realtime
- github.com/supabase/supavisor
- github.com/supabase/pg_graphql
- postgrest.org
- wal-g.readthedocs.io
