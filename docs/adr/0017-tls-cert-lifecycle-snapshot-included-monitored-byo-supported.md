# TLS cert lifecycle: snapshot-included, expiry-monitored, BYO-cert supported, DNS-01 opt-in

NTTS owns the TLS cert lifecycle for custom domains end-to-end: `ntts backup snapshot` includes `/data/caddy` alongside `pg_dump`; an hourly health check populates `_ntts.tls_cert_status` and feeds the F-LOG alert engine for expiry warnings; renewal errors raise their own audit row + ERROR alert; Operators can opt into Bring-Your-Own-Cert per domain with key material stored in Vault; ACME challenge type defaults to HTTP-01 with DNS-01 as an opt-in for environments without port-80 access. Caddy remains the workhorse; NTTS adds the surrounding observability and restore-safety.

## Considered Options

- **Treat `/data/caddy` as an opaque Docker volume the Operator manages.** Rejected: contradicts the appliance promise ("one command backup, one command restore"). LE rate-limits make naive restores expensive.
- **Continuous S3-mirror of Caddy state.** Rejected for v1.0: overengineering for a single-node appliance; certs+state are KB-sized and survive in the snapshot fine.
- **No monitoring, rely on Caddy logs.** Rejected: an Operator who isn't tail'ing Caddy logs finds out at outage time. The alert engine already exists for disk/health; folding TLS expiry into it is cheap.
- **LE-only, no BYO-cert.** Rejected: corporate-PKI requirements are a real procurement blocker for sovereign-EU deployments.
- **DNS-01 as the default.** Rejected: harder Operator setup (DNS API tokens) for the common case. HTTP-01 default, DNS-01 opt-in.

## Mechanism

### Backup integration
- `ntts backup snapshot` produces two artefacts: a `pg_dump` tarball and a `tar` of `/data/caddy`.
- `ntts backup restore` restores both atomically. Caddy on restart picks up the existing cert/state from `/data/caddy` — no re-issuance, no rate-limit risk.
- WAL-G PITR is Postgres-only and does not cover `/data/caddy` — restoring to a point-in-time mid-WAL-stream uses the latest snapshot's Caddy state. Documented limitation.

### Expiry monitoring
- pg_cron `_ntts.check_cert_expiry()` runs hourly. Reads Caddy's local admin API (`http://ntts-edge:2019/pki/ca/local`) or scans `/data/caddy/certificates/` filesystem if the admin API is disabled. Populates `_ntts.tls_cert_status (domain, issuer, not_before, not_after, last_renewed_at, last_renewal_error, source enum('le','byo'))`.
- F-LOG-4 alert engine reads `tls_cert_status`. Default thresholds: WARN at `not_after < now() + 14 days`, ERROR at `< 7 days`, CRITICAL at `< 2 days`.
- Renewal failures (Caddy emits a log line on ACME failure) parsed by a log-tail job → `_ntts.tls_renewal_errors (domain, error_code, error_message, occurred_at)` + ERROR alert raised.

### Bring-Your-Own-Cert
- Studio "Domains" UI: per-domain toggle `auto_tls` (LE) vs `custom`.
- Custom: Operator uploads cert.pem + key.pem (or supplies Vault references). Key material stored as `vault.secrets[tls_key_<domain>]`. Cert chain stored as `vault.secrets[tls_cert_<domain>]` (encrypted at rest; not strictly secret but kept beside the key).
- Studio-backend renders a Caddyfile fragment per domain at boot and on rotation. Key material is materialised to tmpfs (`/var/run/ntts/tls/<domain>/`) for Caddy to read, never written to persistent disk in plaintext.
- Renewal is Operator-managed. NTTS only monitors `not_after` and alerts.
- Hybrid is supported: a deployment can mix LE and BYO domains.

### ACME challenge type
- Default: HTTP-01 (Caddy default, requires Port 80 open).
- Opt-in DNS-01: Studio Domains UI exposes "Use DNS-01" with provider catalog (Cloudflare, Route53, DigitalOcean, Hetzner, …). Provider API tokens stored in Vault; Caddy DNS plugin reads tokens via env injected by Studio-backend.
- Per-domain choice: deployment can mix HTTP-01 and DNS-01.

### CLI surface
- `ntts tls list` — show all domains, source (LE/BYO), expiry, last renewal.
- `ntts tls set <domain> --source le | --source byo --cert FILE --key FILE`
- `ntts tls challenge set <domain> --type http01 | --type dns01 --provider <name>` (DNS-01 prompts for API token; lands in Vault).
- `ntts tls renew <domain>` — force renewal for LE (debug/recovery). No-op for BYO.

### Role gates
- Edit/rotate/delete domain TLS config: admin + MFA.
- Reveal BYO key material: admin + MFA re-auth (Vault reveal pattern from ADR-0008).
- View status (without keys): admin + developer.

## Consequences

- Backup snapshots grow by a few hundred KB to a few MB per domain — negligible.
- An Operator who restores a snapshot to a different hostname must still update DNS for that hostname; certs are domain-bound. NTTS does not solve DNS, only the cert-state-on-restore problem.
- BYO-cert workflow gives Operators a path through corporate-PKI requirements without changing NTTS core. Renewal stays Operator's responsibility — the monitor + alert provides the safety net.
- The pattern (Vault-stored secrets, tmpfs-materialised at boot, NOTIFY-driven rotation) is now reused for Function Secrets (ADR-0007), OAuth Providers (ADR-0014), Wrappers (ADR-0016), and TLS BYO-keys. A shared "secret-materialiser" abstraction in studio-backend is worth extracting.
- DNS-01 opens a path for Operators behind firewalls (no Port 80 to the public internet) but introduces a new credential class (DNS provider API tokens). Audit-logged as platform config changes.
- The alert engine carries one more signal source; thresholds are documented as `tls.expiry.warn_days`, `tls.expiry.error_days`, `tls.expiry.critical_days` in `_ntts.config`.
