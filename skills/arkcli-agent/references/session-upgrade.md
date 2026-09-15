# Upgrade an Existing Session

Use `agent session upgrade` to change the model, Agent version, or runtime configuration of an **existing Session**. Updating the Agent resource does not automatically update its old Sessions; `+iterate` creates a new Session and is not a substitute. Use `session update` for title/tags only. Creation-time overrides are described in `session-files.md`.

## Command and Input

Consult [model parameter metadata](model-config.md) before configuring or changing runtime parameters. Metadata guides supported values; it does not replace upgrade baseline semantics or add client-side business validation.

```bash
arkcli agent session upgrade <session-id> \
  --model <selected-model-id> \
  --thinking enabled \
  --reasoning-effort low \
  --dry-run --format json
```

```bash
arkcli agent session upgrade <session-id> \
  --agent-version 4 \
  --agent '{skills: [], mcp_servers: []}' \
  --environment '{config: {env: {LOG_LEVEL: debug}, packages: {pip: [pytest]}}}' \
  --compact \
  --dry-run --format json
```

| Flag | Meaning |
| --- | --- |
| `<session-id>` | Required; preserves the existing Session rather than creating one |
| `--file <path>` | Local JSON/YAML request file; no implicit stdin or `--file -` support |
| `--agent <object>` | Inline JSON/YAML or `@file`; fields: `type/id/version/model/system/tools/mcp_servers/skills/multiagent/display_name` |
| `--environment <object>` | Inline JSON/YAML or `@file`; fields: `type/id/config` |
| `--agent-version <int>` | Overrides `agent.version`; omitted or 0 preserves the current snapshot, not the latest version |
| `--model <id-or-object>` | An ID replaces only `agent.model.id`; an object or `@file` replaces the model object in the request |
| `--speed/--thinking/--reasoning-effort/--service-tier` | Overrides the corresponding snake_case model field, preserving explicit empty strings |
| `--vault-ids <array>` | JSON/YAML array or `@file`; refresh existing bindings, not add/remove them |
| `--initial-events <array>` | JSON/YAML array or `@file`; optional or `[]`; the backend accepts only one fixed `/compact` event when nonempty |
| `--compact[=false]` | true builds the fixed `/compact` event; false sets `initial_events: []`; conflicts with `--initial-events` |

This uses data-plane `POST /api/v3/sessions/{id}/upgrades` and **snake_case**, not TOP CamelCase or creation-time `AgentWithOverrides`. Missing type values become `agent_with_upgrades` / `environment_with_upgrades`; explicit types are forwarded unchanged. IDs may be omitted to keep current bindings. Supplied Agent/Environment IDs must match the Session and cannot switch its bindings.

Precedence: load `--file`, replace its entire agent/environment object when the respective flag is present, then apply version/model leaf flags. Event/Vault flags replace the corresponding arrays. The CLI preserves omitted values, `null`, `[]`, and empty strings. Apart from defaulting missing discriminators, it does not look up or merge remote snapshots or enforce business allowlists; report server errors without inventing unsupported fields.

## Configuration Semantics

- Supply at least one of `agent/environment/vault_ids`; initial events alone are not an upgrade target. This API does not expose top-level `title/tags/resources/checkpoint_id/environment_id`.
- Without a version, patch the current Session Agent snapshot and preserve unprovided fields. With a positive version, start from that published version and apply explicit overrides; previous Session overrides are not automatically retained. Resolve an actual published version number when the user asks for the latest; neither omission nor the string `latest` means latest.
- `tools/mcp_servers/skills` replace entire arrays; `[]` clears them and omitted/null preserves the baseline. No default tools or skills are injected.
- Model fields: `id/speed/thinking/reasoning_effort/service_tier/provider/protocol/base_url/headers`. **Unlike creation-time overrides, upgrade supports model connection fields and does not filter them.** The backend resolves model identity and effective protocol/defaults. Use the selected model's metadata for runtime settings; never inject a provider, URL, headers, or speed by assumption.
- Cloud networking replaces the policy; `config.env` merges by key and packages merge by package name, with supplied values/specs winning. Empty env/packages do not clear existing entries. Explicit setup_script replaces it, including an empty string. An environment ID alone does not reload the Environment resource's latest configuration; supply the desired config explicitly.
- Self-hosted Sessions can upgrade their Agent, but cannot switch environment type or configure customer-owned sandbox networking/packages/env/setup_script. User TOS cannot be changed by upgrade.
- Vault IDs must equal the current binding set, ignoring order/duplicates. `[]` is valid only when already unbound. This refreshes credentials without changing bindings; do not supply credential secrets.

## Execution and Verification

1. Read the Session to identify current bindings/configuration and state. The server requires idle and rejects terminated/upgrading Sessions or a previous turn that has not normally ended. The CLI does not preempt these business checks or retry business errors.
2. Use leaf-local `upgrade ... --dry-run --format json`. It is an offline plan, not model/resource validation. Explain the changes and possible sandbox preparation/compaction effects, obtain explicit confirmation, then execute without dry-run. There is no `--yes` flag on this command.
3. Execution submits once and returns the actual Session object; it does not return a separate upgrade task or wait for completion. `upgrading` means accepted/in progress, not that the new model is already effective. Do not invent an `/upgrades/{upgrade_id}` query command.
4. Separately read the Session until it leaves upgrading and verify model ID, version, runtime settings, and target configuration. Idle alone does not prove success. Diagnose mismatched configuration with events or `+debug`. After an ambiguous response/timeout, read back before considering another upgrade.
5. Do not send messages, delete, interrupt, or repeat an upgrade while upgrading. If compact was requested with upgrade, do not send another compact. After configuration verification, obtain test authorization before sending a minimal inference message.

Use the current product/profile API Key and project; an explicit base URL needs a paired explicit API Key. Standard Authorization/API Key fields are redacted in preview, but arbitrary environment variables/custom headers are not guaranteed to be secret-free. Never disclose real secrets in conversation, logs, or reports.
