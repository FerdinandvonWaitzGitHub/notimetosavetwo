Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Supavisor (Elixir fork) in compose; reachable at `:6543` if Operator opts in via `NTTS_EXPOSE_POSTGRES=true`. Transaction + session pool modes. Same auth path as direct connections (App User JWT or service-role). Operator can scale Supavisor independently of Postgres. This slice covers single-node operation only; the read/write-splitter logic for Cluster Mode lands in slice 65.

## Acceptance criteria

- [ ] `psql -h <host> -p 6543 -U postgres -d postgres` connects (§6 #16)
- [ ] Transaction mode + session mode both selectable per connection
- [ ] Auth via JWT or service-role mirrors direct-connection behavior
- [ ] Capacity-target: pooler sustains 100 RPS through it on 4-vCPU/8-GB single-node VPS
- [ ] `:6543` only exposed when `NTTS_EXPOSE_POSTGRES=true`
- [ ] Supavisor container scalable via compose `--scale` (single-node mode)

## Blocked by

- [01-compose-greens-up.md](./01-compose-greens-up.md)
