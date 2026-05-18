Status: ready-for-human

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

AI assistant (default-off, provider-pluggable) per ADR-0003 + ADR-0020. **Layered prompts**: hardcoded base + Operator-editable per-tool templates stored in `_ntts.ai_assistant_tools (tool_name, base_prompt_version, operator_overlay, updated_at, updated_by)`. **Provider adapters**: OpenAI, Anthropic, Mistral, Ollama, with a uniform `complete(messages, opts)` interface. **Activation**: `NTTS_AI_PROVIDER` env unset → AI features absent in Studio. Setting it to `ollama` with endpoint → SQL editor shows assistant. **Key storage**: provider API key stored as `vault.secrets[ai_provider_key]` (F-VLT-4), read by Studio-backend AI adapter only. **Full audit**: each call lands `_ntts.ai_assistant_log (id, operator_id, tool_name, provider, model, prompt_hash, response_hash, latency_ms, ...)`. HITL because (a) provider adapter shape locks in the prompt-composition contract, (b) per-tool template surfaces in Studio need design review, (c) GDPR posture around prompt+response hashing vs full capture is a judgment call.

## Acceptance criteria

- [ ] `NTTS_AI_PROVIDER` unset → AI features absent in Studio (§6 #21 first half)
- [ ] `NTTS_AI_PROVIDER=ollama` + endpoint → SQL editor shows assistant (§6 #21 second half)
- [ ] OpenAI + Anthropic + Mistral + Ollama adapters implemented + tested
- [ ] Operator can edit per-tool template overlay; base prompt versioned
- [ ] AI provider API key stored in Vault; never logged plaintext
- [ ] Every assistant call lands an `_ntts.ai_assistant_log` row
- [ ] Provider failure → graceful degradation (assistant disabled with banner; no API request errors leaked to Operator)

## Blocked by

- [07a-studio-shell-login-mfa-nav.md](./07a-studio-shell-login-mfa-nav.md)
