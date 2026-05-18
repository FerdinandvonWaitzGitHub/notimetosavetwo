Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

CI compat smoke-test gate (§7 Compat). Runs the matrix of supabase-js (latest stable + last-2-minor) + Swift SDK quickstart against a fresh `compose up` NTTS Deployment. Covers §6 acceptance criteria 1, 3, 13, 14, 15, 16, 17, 25 end-to-end.

## Acceptance criteria

- [ ] `supabase-js` quickstart (signup → login → REST select → storage upload → realtime subscribe) passes against fresh NTTS Deployment
- [ ] Swift SDK quickstart (signup → REST select) passes
- [ ] Last 2 minor versions of supabase-js tested in matrix
- [ ] Test gates on every main-merge + release-PR; non-zero exit blocks merge
- [ ] CI runtime <10 min on standard runner

## Blocked by

- [55-js-sdk-client.md](./55-js-sdk-client.md)
- [56-mobile-sdk-swift.md](./56-mobile-sdk-swift.md)
