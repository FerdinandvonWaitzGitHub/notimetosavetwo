# AI assistant prompt boundary: layered base + per-tool composition, provider-specific adapters

The AI assistant's system prompt is composed of two layers: a hardcoded **base layer** carrying NTTS-context, privacy boundaries, and refusal rules; and a **tool layer** carrying task-specific instructions per feature (SQL NL→SQL, RLS designer, function helper, etc.). The base is shared across every tool — common safety framing lives in exactly one place — while tool-layer templates live in `_ntts.ai_assistant_tools` and are Operator-editable. A per-provider adapter (`openai.ts`, `anthropic.ts`, `mistral.ts`, `ollama.ts`) renders the composed prompt into the provider's native message format. Every call logs prompt + response with version pointers to `_ntts.ai_assistant_log` for audit.

## Considered Options

- **Single system-prompt per provider.** Rejected: forces every tool through the same framing, contextually wrong for RLS design vs SQL translation vs function authoring; quality suffers across all tools.
- **Per-tool prompts without sharing.** Rejected: privacy/safety rules duplicated N times; drift between tools' safety framing inevitable, audit gets harder.
- **Layered, single provider format.** Rejected: a hardcoded format (e.g. OpenAI `messages`) breaks down for Anthropic's `system:` field separation or Ollama's chat template peculiarities. An adapter layer is the right place to handle those.

## Mechanism

### Base layer (hardcoded in studio-backend AI module)
- Versioned constant `BASE_PROMPT_V<N>` in code.
- Content: NTTS identity, hard constraints, refusal rules, output-format placeholder.
- Hard constraints encoded:
  - Schema-only access (table names, column types, policies; never row data).
  - Function code visible; secret values never visible (only secret names).
  - No fabrication of column names — if uncertain, ask.
  - No dynamic SQL when parameterised SQL suffices.
  - Refusal triggers: disable-RLS-without-explanation, exfiltration intent, SECURITY DEFINER without role justification.
- Base edits require a code change + NTTS release (deliberately not Operator-editable; safety-critical).

### Tool layer (DB-stored, Operator-editable)
- Table `_ntts.ai_assistant_tools (id text PK, display_name, base_version int, prompt_template text, default_few_shots jsonb, enabled bool, updated_by, updated_at)`.
- Default tools shipped at v1.0: `sql_assistant`, `rls_designer`, `function_helper`, `schema_namer`, `policy_explainer`, `error_diagnoser`.
- Template includes placeholders: `{schema_snippet}`, `{table_definition}`, `{existing_policies}`, `{few_shots}` — filled by studio-backend before send.
- Operators can edit a tool's template (Studio AI Settings page, admin + MFA); each edit bumps a `tool_version` integer, lands in `studio_audit_write`, and stores the previous template in `_ntts.ai_assistant_tools_history`.

### Transport (streaming) and tool-calling scope

- **Streaming transport:** **WebSocket** between studio-backend and studio-frontend (`/_internal/ws/ai/:tool_id`). Consistent mit dem existierenden Realtime-WS-Stack und dem `types:changed`-WS-Event aus F-MIG-5 — ein einziger Streaming-Transport im Studio. Adapter wandeln Provider-Stream (OpenAI/Anthropic/Mistral SSE; Ollama NDJSON) in NTTS-interne WS-Frames `{type: 'token' | 'tool_call' | 'done' | 'error', payload}`.
- **Tool-calling (provider-native):** **out-of-scope für MVP / v1.0.** Begründung: APIs divergieren stark (Anthropic content-blocks vs OpenAI function_call vs Mistral vs Ollama uneinheitlich). Schema-Stuffing über `{schema_snippet}` + context-aware Tabellen-Filter (nur Tabellen, die der Operator gerade in Studio offen hat) ist für die v1-Tool-Suite ausreichend. Revisit, sobald Ollama Tool-Calling unified unterstützt.

### Provider adapters
- Path: `studio-backend/src/ai/adapters/<provider>.ts`.
- Adapters: `openai`, `anthropic`, `mistral`, `ollama` (per ADR-0003 provider list).
- Each adapter:
  - Accepts a `ComposedPrompt {base, tool, placeholders_resolved, user_message, few_shots}` IR.
  - Renders provider-native format:
    - OpenAI / Mistral: `{model, messages: [{role:'system', content: base+tool}, …few_shots, {role:'user', content: user_message}]}`.
    - Anthropic: `{model, system: base+tool, messages: […few_shots, {role:'user', content: user_message}]}`.
    - Ollama: `{model, messages: [...]}` (OpenAI-compatible shape) — concrete fields may differ per Ollama version; adapter handles.
  - Handles per-provider token-counting estimates, exponential-backoff retries, error mapping to a common `AIError {kind, message, retryable}` type.

### Placeholder filling and privacy
- Pre-call, studio-backend resolves placeholders from current DB state.
  - `{schema_snippet}`: tables/columns/types, no sample data.
  - `{existing_policies}`: RLS policy text.
  - `{few_shots}`: from tool's `default_few_shots` plus operator-added examples.
- An explicit PII scrubber rejects any placeholder value that contains a known PII signature (email, IBAN, common PII patterns) — defensive; in practice placeholders should never carry such values.
- Logs are written **after** placeholder resolution, so the audit reflects what actually went to the provider.

### Audit
- `_ntts.ai_assistant_log (id uuid, operator_id, tool_id, base_version, tool_version, prompt_redacted text, response_redacted text, provider text, model text, tokens_in int, tokens_out int, duration_ms int, called_at timestamptz, error_kind text null)`.
- `prompt_redacted` and `response_redacted` apply a regex-redactor over likely-secret patterns before storage (defence-in-depth).
- Viewable by admin Operator only (F-STU-6 role gate); reveals respect F-STU-7 read-audit pattern.
- Partitioned monthly; retention via F-LOG-1 pattern with a long default (180 days) since AI calls are low-volume and operator-debug-useful.

### Operator-controls
- Per-tool enable/disable toggle in Studio Admin AI Settings.
- Per-tool template edit (admin + MFA).
- Provider selection via `NTTS_AI_PROVIDER` env + provider key in `vault.secrets[ai_provider_key]` (ADR-0008 / F-VLT-4); rotation is a Vault rotation.
- "Test prompt" UI per tool: composes the layered prompt, lets the Operator inspect what would be sent, and optionally fires the call against the configured provider with a mock placeholder set.

## Consequences

- Adding a new provider = one new adapter file + a registration entry; no other code changes. The provider list in ADR-0003 grows here.
- Adding a new tool = one row in `ai_assistant_tools` + a Studio entry-point that calls the AI adapter with the right `tool_id`. No backend code change for the tool itself.
- Base-prompt edits are deliberately friction-heavy (code change + release) — the privacy boundary lives there and we treat it like a config-as-code surface, not a runtime knob.
- `_ntts.ai_assistant_log` grows with usage but is low-volume by nature; partitioned monthly with 180-day retention. Operators who do not enable AI never write rows.
- The `ai_assistant_tools_history` table is intentionally not hash-chained — operational not compliance data — but is sufficient to roll back a prompt edit if a tool's quality regresses.
- Tool-layer templates are Operator-editable: this is power and risk. A misedit can degrade output quality but cannot violate base-layer constraints because the base is composed *after* the tool layer in adapters where ordering matters. Adapters enforce base-precedence.
