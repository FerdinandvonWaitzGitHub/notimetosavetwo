# NoTimeToSave (NTTS) Backend

A self-hosted, single-deployment, EU-sovereign BaaS appliance that targets feature parity with Supabase at the SDK surface. The repo holds the control-plane, studio, CLI, and the docker-compose definition that wires wrapped upstream components together.

## Language

### Identity & access

**Operator**:
A human who administers an NTTS Deployment. Has a row in `_ntts.studio_users` and one of three roles. Not a single shared account — multiple Operators can share one Deployment.
_Avoid_: "admin user" (ambiguous with App User admin), "tenant admin" (NTTS has no multi-tenancy).

**Studio User**:
Synonym for **Operator**. Use "Operator" in prose; "studio_user" only when referring to the DB row.

**App User**:
An end-user of the Operator's application. Lives in `auth.users` (GoTrue's table). Completely separate from Operators — different table, different login surface, different MFA enrolment, different audit trail.
_Avoid_: "customer", "user" (unqualified — always say App User or Operator).

**Role**:
One of `admin`, `developer`, `viewer`. Determines what an Operator can do in Studio. Stored on `_ntts.studio_users.role`.
_Avoid_: "permission", "scope" (loaded terms — Role is the canonical capability boundary).

**Setup Token**:
A single-use, time-limited bootstrap credential printed to control-plane stdout on the first `docker compose up`. Lets the first human claim the initial admin Operator account. Expires 24h or on use.
_Avoid_: "root password", "install key" (Setup Token is not reusable — it disappears after first admin exists).

### Deployment & tenancy

**Deployment**:
One running NTTS stack — one docker compose project, one Postgres, one set of containers. Single-tenant by definition: one Deployment serves one application's App Users.
_Avoid_: "instance" (overloaded with V8 isolate / Docker container), "tenant" (NTTS doesn't have a tenant entity), "environment" (operators run separate Deployments for dev/staging/prod).

**Cluster Mode**:
Optional, off-by-default scaling: read replicas + multi-node Realtime. Engaged explicitly by the Operator; the appliance promise is single-node.
_Avoid_: "HA mode", "production mode" (single-node is a production mode too).

### Backend processes

**Studio-Backend** (`studio-backend`):
The single Node/TS container that owns every server-side responsibility *other than* the wrapped upstreams. Concretely: serves the Studio Frontend's HTTP API, mutates the `_ntts` schema, emits `NOTIFY` events, holds the Docker socket and watches container health, runs the Setup-Token bootstrap, issues Operator-Invites, orchestrates Migrationen and deployments. One process, one container, one Postgres connection pool.
_Avoid_: spinning up a second process called "control-plane". The Control-Plane responsibility lives *inside* studio-backend.

**Control Plane**:
The *architectural concept* of state-mutating, NOTIFY-emitting, orchestration-owning code paths. Not a container, not a process — a role that lives inside [[studio-backend]]. Used in prose to talk about *what kind of work* is happening; never used as a container name.
_Avoid_: "the control-plane container" — there isn't one. Say `studio-backend` for the container, "control-plane responsibility" for the role.

**ntts-typegen**:
Separate Node/TS worker container holding a `LISTEN ntts_typegen`. Generates `_ntts.generated_types` rows (`name='db'` und `name='functions'`) after schema changes. Decoupled from [[studio-backend]] so that long type-gen runs don't block API requests. See [ADR-0012](./docs/adr/0012-type-gen-flow-dedicated-worker-async-versioned.md).

**ntts-egress-proxy**:
Separate container providing the outbound-HTTPS choke-point for Postgres-side wrappers (FDWs). Egress allowlist enforced here. See [ADR-0016](./docs/adr/0016-wrappers-fdw-catalog-driven-with-egress-proxy-and-cost-guardrails.md).

### Edge & trust

**Gateway** (`ntts-edge`):
The single component every external request passes through. Built on Caddy with a custom plugin layer. Owns: TLS termination, JWT validation, request-ID generation, rate-limit, service-role detection, audit hooks. All upstreams (PostgREST, GoTrue, Storage, Realtime, Edge Runtime, Studio backend) sit behind it on an internal Docker network and trust it.
_Avoid_: "proxy", "API gateway" (too generic — say "Gateway" or "ntts-edge"), "Caddy" (Caddy is the implementation base, not the role).

**Trust Boundary**:
The line between the Gateway and the rest of the stack. Outside it: untrusted, JWT-validated, rate-limited. Inside it: trusted, identity carried in internal headers (`X-NTTS-Role`, `X-NTTS-User-Id`, `X-NTTS-Request-Id`), no further JWT checks.
_Avoid_: "secure zone", "DMZ" (network terms — Trust Boundary is the auth/identity term).

**Service Role**:
A privileged role that bypasses RLS, intended for server-side automation (cron-triggered Functions, internal Function-to-Function calls, control-plane operations). Detected by the Gateway from a JWT with `role: service_role`. The service-role secret lives only in the Gateway and in the Operator's env — never in SDKs, frontends, or other containers.
_Avoid_: "admin role", "root" (App User role table has its own meanings).

### Logic tier

**Function**:
A TypeScript module deployed to NTTS's Edge Runtime. Has a name, a Visibility, and zero or more deployed Versions.
_Avoid_: "lambda", "endpoint" (a Function may be private and have no HTTP endpoint).

**Bundle**:
An eszip-compiled, immutable artifact of one Function build. Stored in `_ntts.function_bundles`. Many Bundles can exist for one Function; only one is "active".
_Avoid_: "build", "deployment" (a Bundle is the *thing*; a deployment is the *act* of pointing the active pointer at it).

**Version**:
The active Bundle for a Function — the row pointed at by `_ntts.function_deployments.active_version_id`. Rolling back swaps the pointer to a different Bundle. Last 20 Bundles per Function are retained.
_Avoid_: confusing with semver — Versions here are deploy-pointer history, not semantic versioning.

**Visibility**:
A Function attribute: `public` (callable via `/functions/v1/:name`), `private` (callable only function-to-function or by DB event/cron), `event` (only triggered by pg_net/pg_cron, no direct caller).
_Avoid_: "scope", "exposure".

**Function Secret**:
A named, pgcrypto-encrypted value scoped to one Function. Stored in `_ntts.function_secrets`. Read at user-worker boot, exposed to Function code via `Deno.env.get(name)`. Per-Function — the same value used in N Functions is stored N times.
_Avoid_: "env var" (Function Secret has a database backing and a rotation flow — plain env vars are static config from the Operator's perspective), "vault secret" (Vault is a separate, stronger feature — pgsodium-backed).

**Master Key** (`NTTS_MASTER_KEY`):
The pgcrypto symmetric key that encrypts/decrypts all Function Secrets. Lives only in the Edge Runtime container env. Losing it loses all Function Secrets. Operator-owned, backed up out-of-band — never in `_ntts`.
_Avoid_: "encryption key" (too generic — there are also SAML signing keys, JWT secrets, Caddy TLS certs).

**Vault Secret**:
A pgsodium-encrypted value in the `vault.*` schema, for **SQL-tier consumers**: PL/pgSQL, `wrappers` FDWs, pg_net callers, Studio-backend SQL. Separate store, separate master key (pgsodium server key on the Postgres data volume), separate UI tab from Function Secrets. Functions (TS) do not read Vault.
_Avoid_: collapsing "Vault" and "Function Secrets" into "secrets" — the two are intentionally separate, see [ADR-0008](./docs/adr/0008-three-classes-of-secrets.md).

**System-Bootstrap Secret**:
A value in the Operator's `.env` consumed at container start by Gateway, GoTrue, Edge Runtime, Postgres. Includes `NTTS_MASTER_KEY`, `JWT_SECRET`, `SERVICE_ROLE_KEY`, pgsodium server key path, SMTP credentials. Never in `_ntts`, never in Vault — it's the *root* of trust the other two stores depend on.

**Pre-deploy Pipeline**:
The 8-stage hard-block check that runs before a Bundle can become a Version: tsc → eslint → bundle → test → schema-compat → security-scan → signature → dry-run-deploy. Operators with `developer` or `admin` Role can override with a written reason; the override lands in `_ntts.function_audit`.

### Compatibility

**Drop-in for Supabase**:
The promise that an existing app using `@supabase/supabase-js` runs unchanged against NTTS, with all of REST, GraphQL, Auth, Storage, Realtime, Functions, and the connection Pooler reachable at the same paths Supabase uses. The architectural exclusions in §9 of the PRD are the only documented gaps.
_Avoid_: "Supabase compatible", "Supabase clone" — too loose. "Drop-in" implies SDK-surface parity, not internal architecture parity.

## Relationships

- One **Deployment** is owned by one or more **Operators**.
- One **Operator** has exactly one **Role** within a Deployment.
- One **Function** has zero or one **Version** at a time, and many historical **Bundles**.
- The active **Version** is the **Bundle** that `_ntts.function_deployments.active_version_id` points to.
- Every **Version** change is recorded in `_ntts.function_audit` with the **Operator** who performed it.
- **App Users** (`auth.users`) have no relationship to **Operators** (`_ntts.studio_users`) — they're parallel, independent identity systems.
- Every external request crosses the **Trust Boundary** exactly once, at the **Gateway**.
- The **Service Role** is never carried by App User or Operator JWTs — it is a separate, server-only credential.

## Example dialogue

> **Dev:** "If an **Operator** with `viewer` Role views the **App User** list, does that trigger anything?"
> **Architect:** "Yes — every PII read by an **Operator** writes a row to `_ntts.studio_audit`. We need to defend that to a GDPR auditor."

> **Dev:** "Can a private **Function** be deployed without going through the Pre-deploy Pipeline?"
> **Architect:** "No. Visibility is independent of the Pipeline. Public, private, event — all eight stages run."

> **Dev:** "What if the **Setup Token** is lost before the first admin claims it?"
> **Architect:** "Operator restarts the control-plane container. If no `studio_users` row exists, a new Setup Token is minted and printed."

## Flagged ambiguities

- "user" was used to mean both **App User** and **Operator** — resolved: these are distinct, and we never say plain "user" without qualifier.
- "single operator" in the original PRD (§3) implied one human per Deployment — resolved: a Deployment is single-tenant (one application namespace), but supports multiple **Operators**.
- "instance" was used for V8 isolates, Postgres containers, and Deployments — resolved: drop the word entirely; say isolate / container / Deployment.
- "user-invite" wizard in §4.6 was ambiguous — resolved: there are two distinct flows, **App-User-Invite** (writes to `auth.users`) and **Operator-Invite** (writes to `_ntts.studio_users`, admin-only).
