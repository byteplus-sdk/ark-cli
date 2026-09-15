# Model Runtime Parameter Metadata

Before configuring runtime parameters when creating/updating an Agent or overriding/upgrading a Session, query the target model's metadata. `agent model list` selects models; it does not provide these parameters' complete supported values.

## Query

This API does not provide `epa_tool_model.support`. Use `agent model list --usage tool` to filter version-level metadata from the complete ArkModels catalog; see [tool model discovery](agent.md#discover-tool-models-and-cache-behavior) for parameters and caching. An empty key lookup here does not establish tool-model ineligibility.

```bash
arkcli agent model config <foundation-model-name> --provider ark --format json
arkcli agent model config <foundation-model-name> --key thinking.support_value,thinking.default --key service_tier.support_value,service_tier.default --format json
```

| Input | Meaning |
| --- | --- |
| `<foundation-model-name>` | Required. Use the selected `agent model list` item's `name`, not its versioned `model`, model ID, or `ep-*`. Never derive a name by stripping a version suffix. |
| `--provider` | Defaults to `ark`, matching the frontend. This does not switch accounts or products. Use another provider only when the target model identity is established. |
| `--key` | Repeatable and comma-separated; omitted means all metadata. Keys are passed to the server without a client allowlist. |

This read-only control-plane command uses the current profile and calls `ListManagedAgentModelMetaDatas`. There are no model-version, pagination, or `--dry-run` flags. Output preserves `ResponseMetadata` and the `Result` array; each item contains `FoundationModelName/Provider/Key/Data`. `Data` is a string array; the public API does not return the internal Schema.

If given only a versioned model identifier, match its `model` in `agent model list` and read that item's `name`. Do not use an Endpoint ID as the name. Report server errors without silently switching account, product, or provider.

## Parameter Reference

| Parameter | Meaning | Metadata keys |
| --- | --- | --- |
| `speed` | A model-defined speed mode, available only when supported; not a universal acceleration switch. | `speed.support_value`; the current contract defines no `speed.default`. |
| `thinking` | Thinking mode; some models support values such as `enabled`, `disabled`, or `auto`. | `thinking.support_value`, `thinking.default` |
| `reasoning_effort` | Reasoning effort; some models support values such as `low`, `medium`, or `high`. | `reasoning_effort.support_value`, `reasoning_effort.default` |
| `service_tier` | Model serving tier; some models support values such as `default`, `fast`, or `auto`. This is not CLI plan selection. | `service_tier.support_value`, `service_tier.default` |

These values are illustrative, not a universal enum. Read speed values directly from the target model response as well. `protocol` describes model protocol information; it does not authorize changing the server-managed `Protocol` field.

For example, `Key: "thinking.support_value", Data: ["enabled", "disabled"]` and `Key: "thinking.default", Data: ["enabled"]` declare those supported values and that default for the queried model only.

## Use the Response

- Match `Result[].Key` and read its `Data`. Defaults are usually singleton arrays; the first supported value is not necessarily the default.
- Report missing keys, empty arrays, or inconsistent defaults as returned. Do not invent enums or defaults.
- An unknown model or provider may still return known keys with empty `Data`; an unknown key may return `Result: []`. This query is not a model-existence check.
- Metadata guides configuration; it does not add client-side business validation. Explicit user parameters still follow the create/override/upgrade contract and server validation. Successful persistence does not prove a value is declared supported by metadata.
- Do not inject unrequested parameters or defaults. Omitted upgrade fields retain the documented upgrade baseline semantics; metadata defaults must not overwrite them.
- Query again when changing models. Old model capabilities must not be carried over. A successful metadata query proves neither model activation nor inference availability.

See [Agent](agent.md), [Session overrides](session-files.md), and [Session upgrade](session-upgrade.md) for write locations and merge semantics.
