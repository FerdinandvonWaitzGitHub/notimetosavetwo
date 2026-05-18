Status: ready-for-human

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

**Visual schema designer (ERD)** + **RLS test-sandbox with personas**. ERD renders all `public.*` tables + FK edges; click-to-edit opens table-edit wizard; layout persisted per Operator. **RLS test-sandbox**: define a set of "personas" (App User JWT shapes with claim sets); pick a persona; run any SELECT/INSERT/UPDATE/DELETE against the live schema; see returned rows + which policies allowed/denied each row. Personas persisted in `_ntts.rls_test_personas`. HITL because both surfaces are heavy UX and benefit from layout/interaction design review.

## Acceptance criteria

- [ ] ERD renders all `public.*` tables with FK edges; layout persisted per Operator
- [ ] Table-edit wizard opens from ERD; saves migration file
- [ ] RLS test-sandbox: define persona → run query → see returned rows + per-row policy trace
- [ ] Persona save uses `_ntts.rls_test_personas`
- [ ] Sandbox queries do NOT mutate prod schema (run in transaction + rollback)
- [ ] Capacity-target: ERD handles 1.000 tables in `public.*` (§7) with usable layout

## Blocked by

- [06-type-gen-worker.md](./06-type-gen-worker.md)
