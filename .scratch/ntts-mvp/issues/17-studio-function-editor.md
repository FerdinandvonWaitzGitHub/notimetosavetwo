Status: ready-for-human

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Studio Function editor with live TS check (squiggles in ≤500ms, save blocked in strict mode), schema-aware types pulled from `_ntts.generated_types` (refreshed via WS `types:changed`), and a live Pipeline UI showing each of the 8 stages as they run. Editor uses Monaco. Pipeline UI streams stage progress + stage logs (WS); on hard-block surfaces the failing stage + the reason; offers override form (gated by Role) with reason category + free text. HITL because the UX of the inline-types + live-pipeline + override flow is novel and benefits from a design review before lock-in.

## Acceptance criteria

- [ ] TS errors squiggle in ≤500ms; save blocked in strict mode (§6 #5)
- [ ] Schema-aware types update on next save/debounce after migration
- [ ] Pipeline live UI streams stage progress for all 8 stages
- [ ] On hard-block, override form appears for `admin`/`developer` (hidden for `viewer`)
- [ ] Override form requires both reason category + reason text before Submit enables
- [ ] Stage logs accessible inline per stage
- [ ] Type-fingerprint mismatch warning surfaces in editor

## Blocked by

- [06-type-gen-worker.md](./06-type-gen-worker.md)
- [11-first-deployed-function.md](./11-first-deployed-function.md)
