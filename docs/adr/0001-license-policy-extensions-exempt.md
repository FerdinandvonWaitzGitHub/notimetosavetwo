# License policy: NTTS is Apache 2.0; extensions exempt from the permissive-only rule

**NTTS-distributed code is licensed under Apache License 2.0** (chosen for its explicit patent grant in a stack that handles auth, crypto, and secrets; lines up with Supabase main repo and the wider self-hosted-infrastructure default).

Postgres extensions that the appliance *enables* but does not *redistribute as NTTS code* are exempt from the permissive-only constraint — they ship as separate packages, loaded at runtime. This unblocks PostGIS (GPLv2) and pgroonga (LGPLv2.1), which are required for full Supabase feature parity.

## Considered Options

- **Strict permissive-only including extensions.** Rejected: cuts PostGIS, breaks parity for geo-heavy Supabase apps.
- **Drop the license policy entirely.** Rejected: removes a defensible commitment to operators worried about copyleft contamination.
- **Extensions exempt (chosen).** Operator can disable any GPL extension; NTTS-distributed code itself stays Apache-2.0.

Sub-choice on the NTTS code license itself: Apache 2.0 (chosen) vs MIT vs BSD-3 vs PostgreSQL License. Apache 2.0 won for the explicit patent grant and ecosystem alignment.

## Consequences

- Every source file in the monorepo carries an Apache 2.0 header (or an SPDX `Apache-2.0` identifier).
- `LICENSE` at repo root is the Apache 2.0 text; `NOTICE` lists wrapped components and their licenses.
- The full/slim Docker image split ([ADR-0002](./0002-single-node-default-cluster-optional.md) area) becomes a license boundary too: `ntts-postgres:slim` can be marketed as "permissively-licensed stack" for operators with strict copyleft policies.
