Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Two internal HMAC keys per §4.10: `hmac_f2f` (F2F-caller → `edge-runtime:9000/_internal/invoke`) and `hmac_gateway_realtime` (ntts-edge `X-NTTS-Auth` → supabase-realtime). On first `up`, studio-backend generates two 256-bit CSPRNG keys, stores them in `_ntts.platform_secrets[hmac_f2f|hmac_gateway_realtime]` pgcrypto-encrypted with `NTTS_MASTER_KEY`, and materialises clear-keys to `/var/run/ntts/hmac/{f2f,gateway-realtime}.key` on tmpfs (mode `0400`). Consumer containers mount this directory read-only. Hot-reload: `NOTIFY ntts_hmac_rotated` → studio-backend re-materialises → consumers' Inotify-watch picks up. Rotation via `ntts admin rotate-internal-hmac --kind {f2f|gateway-realtime}` (MFA + reason + 30s cool-down) with a 60s dual-key-verify window before old key is deleted.

## Acceptance criteria

- [ ] Both HMAC keys generated as 256-bit CSPRNG on first `up`; stored in `_ntts.platform_secrets` pgcrypto-encrypted
- [ ] Tmpfs materialisation at `/var/run/ntts/hmac/{f2f,gateway-realtime}.key` with mode `0400`
- [ ] Consumer containers (edge-runtime, ntts-edge, realtime) mount `/var/run/ntts/hmac` read-only
- [ ] `ntts admin rotate-internal-hmac --kind {f2f|gateway-realtime}` requires MFA + reason + cool-down
- [ ] 60s dual-key-verify window: old + new keys both accepted; after window old key deleted from `platform_secrets`
- [ ] `NOTIFY ntts_hmac_rotated` fires; consumers re-load without restart
- [ ] Both rotations are the last step of every `ntts upgrade` (F-UPG-3 hook in this slice or wired in slice 60)

## Blocked by

- [01-compose-greens-up.md](./01-compose-greens-up.md)
