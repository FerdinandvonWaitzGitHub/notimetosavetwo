Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Per-domain BYO-cert toggle (LE vs custom). Custom key+cert stored in Vault (slice 45); studio-backend materialises them to tmpfs for Caddy on boot + on rotation NOTIFY. Operator-managed renewal (no auto-renew for BYO).

## Acceptance criteria

- [ ] `ntts tls set example.com --byo --cert cert.pem --key key.pem` uploads cert+key to Vault and configures Caddy
- [ ] Cert+key materialised to tmpfs (mode `0400`) on boot + rotation
- [ ] Toggle from LE to BYO per domain works without downtime
- [ ] Expiry monitoring (`_ntts.tls_cert_status`) tracks BYO certs same as LE
- [ ] No auto-renewal attempts for BYO certs

## Blocked by

- [48a-tls-le-http01-backup.md](./48a-tls-le-http01-backup.md)
- [45-vault-sql-tier-secrets.md](./45-vault-sql-tier-secrets.md)
