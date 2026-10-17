# BYOK Models and Request Credentials

This release covers Direct access only: the MA model gateway calls the configured upstream HTTPS API. The provider must be configured for Direct access in the target MA environment. Obtain its name from the user or verified service configuration; do not infer availability from its name or reuse a staging test provider as a production default. There is no CLI egress-mode switch.

Use this path only when the user explicitly chooses a third-party model and provides its connection settings. Use a verified Platform profile in the current BytePlus product, not a Coding Plan profile; select it with invocation-scoped `--profile` when needed. See [Authentication and Profiles](../SKILL.md#authentication-and-profiles). `Model.BaseUrl` addresses the upstream model; CLI `--base-url` instead changes the MA data-plane destination.

## Model Configuration

Keep the two connections separate: `--env` selects the MA control plane and default data plane, while `Model.BaseUrl` selects the upstream. MA authentication uses a compatible Platform profile or an explicit `ARK_API_KEY` / `--api-key`; control-plane operations still need login. Upstream authentication uses `--model-header-env`, not the global API key. Environments may differ when explicitly requested; switching `--env` does not rewrite the upstream URL or guarantee that the profile key works in that MA environment. CLI `--base-url` is not an upstream setting.

Agent create/update accepts an ID or inline JSON/YAML object with `--model`; use `--file` for a complete request. Session creation uses `--agent-overrides`, while existing Sessions use `session upgrade --model`. Upgrade emits snake_case; other paths emit the TOP names below. Common aliases are normalized without rewriting Header keys or values.

| Field | Type and meaning |
| --- | --- |
| `Id` | Selected model identifier; do not invent or replace it |
| `Provider` | Provider name; omitted or `ark` keeps Ark model activation checks; other values skip Ark lookup/activation, but not the MA service activation gate |
| `Protocol` | Provider protocol, for example `openai_chat_completions`, `openai_responses`, `anthropic_messages`, or `ark_responses`; server-validated |
| `BaseUrl` | Full HTTPS model API URL from the user or verified configuration; the gateway does not append a path. For Chat Completions, include the actual upstream path, for example `/v1/chat/completions` |
| `Headers` | String-to-string map of persistent ordinary headers, not secrets |
| `TokenLimits.ContextWindow` | int64 context-window token limit |
| `TokenLimits.MaxInputTokenLength` | int64 maximum input tokens |
| `TokenLimits.MaxOutputTokenLength` | int64 maximum output tokens |
| `InputModalities` | Array of supported input modality strings; does not authorize enabling multimodal tools |
| `Speed/Thinking/ReasoningEffort/ServiceTier` | Optional runtime settings; see [model-config.md](model-config.md). Ark metadata is not authoritative for arbitrary third-party models |

Omitted optional fields stay absent. Explicit empty strings/maps/arrays and zero token limits are preserved; the API decides whether clearing or a value is valid. Do not inject defaults or a client business allowlist.

Custom Direct providers require all three TokenLimits as positive integers, not ContextWindow alone. Use verified upstream specifications. If the upstream is explicitly an Ark model, query its exact name/version metadata in the upstream environment: `context_window`, `max_input_tokens`, and `max_completion_tokens` map to the three fields above. Missing values are not zero; do not borrow another model's limits.

```bash
arkcli agent agent create \
  --name byok-demo \
  --model '{"Id":"<model-id>","Provider":"<provider>","Protocol":"openai_chat_completions","BaseUrl":"https://model.example/v1/chat/completions","TokenLimits":{"ContextWindow":131072,"MaxInputTokenLength":126976,"MaxOutputTokenLength":4096},"InputModalities":["text"]}' \
  --dry-run --format json
```

Replace sample values with verified settings. Confirm before executing and read the created Agent back. Preview does not establish upstream availability.

Agent precedence: file, then explicit `--model` replaces the request Model, then explicit runtime flags. Update without `--version` still reads the current version; it does not merge the entire old model into the new request. Updating an Agent does not update existing Sessions: explicitly upgrade them. Override/upgrade forwards supplied fields; the server defines omission and clearing semantics.

## Request-Only Credentials

Supported commands: `agent session events send`, `agent session compact`, and `+new session`. Repeat `--model-header-env HeaderName=ENV_NAME` as needed. The variable contains the full header value; no `Bearer` prefix is added.

```bash
# Let the user or secret manager set MODEL_AUTH securely; never print it.
arkcli agent session events send <session-id> \
  --text 'hello' \
  --model-header-env Authorization=MODEL_AUTH \
  --dry-run --format json
```

- Preview shows references without reading environment values. Send and compact support preview; `+new session` does not, so use read-only checks and explicit confirmation.
- Execution reads the variables. MA Authorization comes from an explicit MA key or the active Platform profile; model credentials use separate `X-User-Model-Header-*` headers. Supply the upstream mapping on each send/compact invocation; Sessions do not persist it.
- Only event POST requests receive these headers, never reads, SSE, uploads, stored config, or request bodies. A serial event batch reuses the invocation's credentials and does not replay accepted events.
- Duplicate case-insensitive header names, missing/empty variables, and invalid HTTP header values fail before effects. Credential-bearing requests do not follow redirects.
- Never place `Authorization`, `X-Api-Key`, or `Api-Key` in persistent Model.Headers. The backend checks static/dynamic collisions; do not bypass this with static headers.
- Credential propagation for upgrade initial_events is unconfirmed; upgrade has no such flag. Read back the upgraded configuration, then separately compact only if explicitly requested.
- Never disclose secrets in conversation, logs, exports, or reports. Creation or send acceptance does not prove inference success; an authorized test must verify a real reply and terminal events.

## Verification and Troubleshooting

- Use `events send ... --stream --format jsonl`, or omit format, to await a reply. Streaming does not accept `--format json`. Verify the reply, `end_turn`, and exit 0.
- An intermediate model request with `is_error=true` may recover through backend retries. Record attempts and the final outcome; do not resend accepted messages. Final `retries_exhausted` produces exit 5 / `agent_execution_failed`.
- An MA authentication error on message submission concerns the MA key/environment, not necessarily the upstream key. For model failures, check the provider configuration in the target MA environment, full URL, protocol, and upstream credentials. A generic provider error does not prove a 401, rate limit, or network fault; retain the Session ID and timestamp for server logs. An authorized direct upstream probe does not replace MA end-to-end verification.
