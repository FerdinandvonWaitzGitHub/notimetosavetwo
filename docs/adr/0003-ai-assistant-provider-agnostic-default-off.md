# AI assistant: provider-agnostic, default off

The Studio AI assistant (SQL NL→SQL, etc.) ships **disabled by default** and is configured by the operator via `NTTS_AI_PROVIDER` + provider key. Supported providers: OpenAI, Anthropic, Mistral (EU), Ollama (self-hosted). No NTTS-hosted inference, no default cloud provider, no NTTS-side API key.

## Considered Options

- **Bake in OpenAI as default.** Rejected: contradicts "EU-sovereign, GDPR clean" — operator who hasn't read the docs ships PII to US-hosted LLM by accident.
- **Drop AI assistant entirely.** Rejected: Supabase has it, parity goal in scope.
- **Provider-agnostic, default off (chosen).** Operator decides if and where. Local Ollama path exists for fully sovereign setups.

## Consequences

Studio must show a clear "AI features disabled — configure provider" affordance when the env var is unset, otherwise users hunt for a missing button. The abstraction layer (prompt formatting, response parsing) is a real engineering item — not free.
