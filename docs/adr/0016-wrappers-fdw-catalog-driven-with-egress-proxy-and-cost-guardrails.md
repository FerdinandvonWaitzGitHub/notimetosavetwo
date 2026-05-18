# Wrappers/FDW: catalog-driven Studio UX, egress proxy sidecar, per-wrapper cost guardrails

NTTS exposes the Supabase `wrappers` extension (plus `postgres_fdw`) through a Studio UI driven by a versioned catalog manifest — the same pattern as OAuth providers (ADR-0014). Each Wrapper entry declares the server-options, user-mapping-options, foreign-table templates, and the egress hosts the Wrapper needs to reach. Foreign-table queries are routed through a dedicated `ntts-egress-proxy` sidecar that enforces an Operator-curated allowlist of outbound hosts and logs every connection. Per-wrapper cost guardrails (statement timeout, max result rows, monthly query-count cap) limit blast radius from cost-bomb queries or credential leaks. Credentials live in Vault (consistent with ADR-0008's three-class secret model); foreign-table DDL is synthesised by studio-backend, never authored by hand in the catalog-driven flow. Role gates mirror OAuth and Function-Secrets: admin-only for edit/test/delete with MFA-gated Vault reveal; developer can view configs without secrets.

## Considered Options

- **Pure-SQL operator workflow.** Rejected as primary: contradicts the "Studio-first appliance" positioning. Acceptable as a fallback for power users via Studio SQL editor; advanced.
- **No egress enforcement.** Rejected: EU-sovereignty positioning becomes a documentation claim with no teeth if a Wrapper misconfig exfiltrates EU PII to a US endpoint.
- **iptables-based egress allowlist on Postgres container.** Rejected: hostname-based filtering via iptables is operationally fragile (DNS timing, CDN IP rotation). Proxy is the standard answer.
- **CLI-only (IaC) workflow.** Rejected as primary: discovery is hostile ("what Wrappers exist?"); UI for setup, file for GitOps.
- **Cost-guardrails-only (no egress block).** Rejected as the only line of defence: it bounds damage from misuse but does not satisfy the "data does not leave the perimeter" invariant.

## Mechanism

### Catalog
- **Source.** `/etc/ntts/wrappers.catalog.json` shipped in the NTTS image, version-bumped per release.
- **Per-entry schema:** `id, display_name, kind ('wrappers'|'postgres_fdw'), server_options[{name,type,required,label,help}], user_mapping_options[{...}], foreign_table_templates[{name, columns[], options}], setup_guide_url, requires_egress_to[host_patterns]`.
- **Custom entries.** Operator-defined custom Wrappers are not supported (unlike OAuth) — Wrappers require backend extension code in Postgres; catalog growth happens via NTTS releases.

### Configuration storage
- **Table.** `_ntts.wrappers_config (id uuid PK, wrapper_id text, server_name text unique, server_options jsonb, user_mapping_vault_ref text (FK → vault.secrets.name), foreign_tables jsonb, guardrails jsonb, enabled bool, last_tested_at timestamptz, updated_at, updated_by)`.
- **Credentials in Vault.** Never inline in `wrappers_config`. The row stores a Vault-reference (`vault.secrets[name]`); DDL synthesis pulls the decrypted value at apply-time via the SECURITY DEFINER wrapper function (ADR-0008).
- **DDL synthesis.** studio-backend renders `CREATE SERVER`, `CREATE USER MAPPING`, `CREATE FOREIGN TABLE` statements from the row + catalog entry; runs them. Updates diff-and-replace foreign-table objects.

### Egress proxy
- **Container.** `ntts-egress-proxy` joins the compose stack (single-node-friendly per ADR-0002). Minimal HTTP CONNECT proxy (Squid or a ~200-line Go binary).
- **Allowlist.** `_ntts.egress_allowlist (host_pattern, port, added_by, added_at, reason, wrapper_id text null)`. Proxy LISTENs for live updates.
- **Postgres container env.** `https_proxy=http://ntts-egress-proxy:3128`, `http_proxy=…`, `no_proxy=localhost,127.0.0.1,*.local,*.<compose-network>`.
- **Auto-allowlist on Wrapper enable.** Catalog `requires_egress_to` entries auto-INSERT into `egress_allowlist` with `reason='wrapper:<id>'`. Removed on Wrapper delete (after confirming no other Wrapper uses the same host).
- **Audit and visibility.** Every Proxy connection logged to `_ntts.egress_log (id, host, port, bytes_in, bytes_out, started_at, duration_ms, denied bool, source_wrapper_id text null)`. Partitioned monthly. Studio "Egress" tab shows active allowlist rules, recent connections, denied-attempt counts. CRITICAL alert (F-LOG-5) when denied-rate exceeds threshold.

### Cost guardrails
- **Per-wrapper config in `wrappers_config.guardrails`:** `{statement_timeout_ms (default 30000), max_result_rows (default 10000), monthly_query_count_cap (nullable)}`.
- **Enforcement.** Studio creates a wrapping SQL function per foreign table (`_ntts.wrapper_safe_select_<table>(...)`) that applies `SET LOCAL statement_timeout`, `LIMIT` injection, and increments `_ntts.wrappers_usage(wrapper_id, month, query_count)`. App code can either call the wrapping function or query the foreign table directly — direct queries bypass guardrails by design (Operator power-user opt-out per-query).
- **Monthly reset.** pg_cron job rotates `wrappers_usage` partitions and resets monthly counters on month boundary.

### Role gates
- **Edit, test, disable, delete:** admin only.
- **Reveal Vault credential used by Wrapper:** admin + MFA re-auth, audit row in `studio_audit_read`.
- **View Wrapper config (without secrets):** admin + developer.
- **SELECT from foreign tables in app code:** standard Postgres GRANT, default to `service_role` only; admin can extend GRANTs explicitly per foreign table.

### IaC export/import
- `ntts wrappers export > wrappers.toml` produces per-wrapper config with credential values replaced by `${VAULT_REF:name}` placeholders; safe to commit.
- `ntts wrappers apply -f wrappers.toml` resolves Vault references at apply, diffs against current config, applies changes including foreign-table DDL.

### Test flow
- `ntts wrappers test <wrapper_id>` (admin + MFA): runs a Wrapper-specific smoke query (catalog declares it, e.g. `SELECT 1 FROM stripe.products LIMIT 1`). Result in `_ntts.wrappers_test_log`. Enable gate soft-warns if `last_tested_at` older than 30 days.

## Consequences

- A new container (`ntts-egress-proxy`) joins the compose stack. Operationally tiny but a new failure point: if Proxy is down, Wrappers and any Postgres outbound (pg_net, AI-provider calls if EU-routed) stop. Health-check + auto-restart per F-LOG-4 alert engine.
- Wrappers ship with NTTS releases; users wanting Wrapper-X that NTTS hasn't catalogued must wait for a release or write raw SQL bypassing the Studio path.
- Egress audit (`egress_log`) is operational data, partitioned and retained via F-LOG-1 pattern; not hash-chained. Operators wanting compliance-grade evidence of "data left to host X" mirror via `NTTS_AUDIT_DRAIN` if needed (currently `egress_log` is outside the audit-drain scope; can be extended in a future release).
- Cost guardrails are opt-in for queries: power-users querying foreign tables directly bypass them. This is intentional — guardrails are training-wheels for the catalogued path, not a hard ceiling. Operators wanting hard ceilings restrict GRANTs to the wrapping functions only.
- `postgres_fdw` (Postgres→Postgres) participates in the same flow but typically targets internal/private-network Postgres instances; the Operator usually adds the target host to the egress allowlist explicitly.
- The pattern (catalog + Studio + IaC + Vault credentials + audit) is now reused three times (Function Secrets, OAuth Providers, Wrappers). A shared "catalog-driven config" abstraction in studio-backend is worth extracting once a fourth use case appears.
