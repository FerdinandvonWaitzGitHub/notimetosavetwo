Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

TLS termination at Gateway via Caddy with auto-LE HTTP-01 (F-GW-1, ADR-0017). Certs in volume `/data/caddy`. **Backup**: `ntts backup snapshot` tars `/data/caddy` alongside `pg_dump`; restore brings certs back without re-issuance (avoids LE rate-limit). **Monitoring**: pg_cron `_ntts.check_cert_expiry()` hourly populates `_ntts.tls_cert_status`; alert WARN 14d / ERROR 7d / CRITICAL 2d before expiry. Renewal failures → `_ntts.tls_renewal_errors` + ERROR. CLI: `ntts tls list/set/renew`.

## Acceptance criteria

- [ ] Custom-domain `ntts tls set example.com` provisions LE cert via HTTP-01
- [ ] `/data/caddy` included in `ntts backup snapshot`; restore reuses certs (no re-issuance)
- [ ] `_ntts.tls_cert_status` populated hourly by `check_cert_expiry()`
- [ ] Alert escalation 14d/7d/2d works against a force-aged cert
- [ ] Renewal failure logged + ERROR alert + does not block traffic on still-valid cert
- [ ] `ntts tls list/renew` work

## Blocked by

- [01-compose-greens-up.md](./01-compose-greens-up.md)
