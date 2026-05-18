Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Auth hooks (F-AUTH-7) fire HTTP calls to a designated Function (configured per hook type: before-sign-in, after-sign-in, custom-JWT-claims, etc.). Transport via the internal HMAC-protected endpoint (slice 20, `as: 'service'`). Hook configuration in `_ntts.auth_hooks (hook_type, function_id, enabled_at)`. GoTrue fork-patch adds the HTTP-out call to the configured Function for each hook point.

## Acceptance criteria

- [ ] `auth_hooks(hook_type='custom-jwt-claims', function_id=<id>)` causes new JWTs to include claims from the Function's response
- [ ] `before-sign-in` hook can deny sign-in by returning a 4xx
- [ ] `after-sign-in` hook fires async; sign-in not delayed
- [ ] Hook target Function executed with `X-NTTS-Role: service_role`
- [ ] HMAC validation enforced on the hook transport
- [ ] Disabling a hook removes the call without GoTrue restart

## Blocked by

- [11-first-deployed-function.md](./11-first-deployed-function.md)
- [20-function-to-function-call.md](./20-function-to-function-call.md)
