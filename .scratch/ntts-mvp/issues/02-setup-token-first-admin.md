Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

On first `compose up` with empty `_ntts.studio_users`, studio-backend mints a single-use Setup Token, prints it to stdout, and (if `NTTS_SETUP_TOKEN_FILE` set) also writes to a `0600` file. `/setup?token=…` claims the first admin Operator: writes a `studio_users` row with role `admin`, enrols MFA TOTP, issues an Operator JWT. Token expires 24h or on consume; post-state, the token file is overwritten in-place with `CONSUMED` / `EXPIRED` (not unlinked). CLI `ntts login` browser flow completes against this Operator. Audit row written to `_ntts.studio_audit_write` for the bootstrap.

## Acceptance criteria

- [ ] Fresh `compose up` prints Setup Token to studio-backend stdout exactly once
- [ ] `NTTS_SETUP_TOKEN_FILE=/path` writes token to file with mode `0600` (container user); post-consume the file is overwritten to literal `CONSUMED`
- [ ] `/setup?token=<token>` flow creates an `admin` Operator + enrols MFA TOTP + issues JWT
- [ ] Second `/setup?token=<token>` use is rejected
- [ ] If no `studio_users` row exists, restarting studio-backend mints a fresh Setup Token
- [ ] `ntts login` against the new Operator completes; refresh-token written to `~/.config/ntts/credentials.toml` (chmod 600)
- [ ] `_ntts.studio_audit_write` carries the bootstrap row

## Blocked by

- [01-compose-greens-up.md](./01-compose-greens-up.md)
