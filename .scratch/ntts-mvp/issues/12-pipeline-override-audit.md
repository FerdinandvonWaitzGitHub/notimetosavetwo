Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

When any Pipeline stage hard-fails, Operators with `developer` or `admin` Role can override per §10-Q8: enum `override_reason_category` (`bug-hotfix | test-flake | external-dependency | urgent-business | infrastructure-issue | migration-required | other`) plus free-text `override_reason_text` 20–1000 chars (both Pflicht). Studio Submit-Button disabled until both set. Override lands `_ntts.function_audit` row with `operator_id`, `failed_stage`, `override_reason_category`, `override_reason_text`, `bundle_id`, `pipeline_run_id`. CLI: `ntts fn deploy --override --reason-category <enum> --reason-text "<text>"`.

## Acceptance criteria

- [ ] Failed test stage blocks deploy (§6 #6 first half)
- [ ] Override with reason succeeds; audit row written with both reason fields (§6 #6 second half)
- [ ] Submit button disabled in Studio until both reason fields valid
- [ ] CLI accepts `--override --reason-category <enum> --reason-text "<text>"`; rejects if either missing
- [ ] Audit row carries `operator_id` of the overriding Operator (§6 #18)
- [ ] Viewer cannot override (UI hides; API returns 403)
- [ ] Free-text length validated (20–1000 chars)

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
- [10-audit-chain-integrity.md](./10-audit-chain-integrity.md)
