Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

DNS-01 ACME challenge opt-in per domain with provider catalog (Cloudflare, Route53, DigitalOcean, Hetzner, …). API tokens for DNS providers stored in Vault. CLI: `ntts tls challenge set <domain> --type dns-01 --provider cloudflare`.

## Acceptance criteria

- [ ] `ntts tls challenge set example.com --type dns-01 --provider cloudflare` configures DNS-01 for that domain
- [ ] API tokens stored in Vault (reveal admin + MFA)
- [ ] Supported provider list visible in `ntts tls challenge list-providers`
- [ ] LE issues wildcard cert via DNS-01 when configured
- [ ] Falls back / errors gracefully if provider API token invalid

## Blocked by

- [48a-tls-le-http01-backup.md](./48a-tls-le-http01-backup.md)
- [45-vault-sql-tier-secrets.md](./45-vault-sql-tier-secrets.md)
