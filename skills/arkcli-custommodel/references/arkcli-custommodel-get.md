# arkcli models custommodel get

Retrieve details or poll model status.

## Usage

```bash
arkcli models custommodel get <id> [flags]
```

## Examples

```bash
# Retrieve full details
arkcli models custommodel get cm-xxxxx

# Retrieve only key fields to avoid a large JSON response
arkcli models custommodel get cm-xxxxx --transform id,name,active_endpoints,artifact_types

# Retrieve only status for polling
arkcli models custommodel get cm-xxxxx --transform status
```

## Flags

| Parameter | Required | Type | Description |
|---|---|---|---|
| `<id>` | Yes | string | Custom model ID in `cm-xxxxx` form |
| `--transform` | No | string list | Field allowlist. Accepts comma-separated values such as `id,name` or repeated flags. Select `CustomModel` fields such as `id`, `name`, `status`, or `foundation_model`, plus `active_endpoints` or `artifact_types`. This command's `--transform` selects a field subset; it is **not** the global GJSON expression. |

## Understanding `--transform`

The `--transform` flag on `custommodel get` is a command-specific field allowlist that reduces extra queries and response size. It is not the root command's GJSON-style `--transform` and does not support nested expressions such as `items.0.id`.

Common selections:

- `id,name,status,foundation_model`: retrieve basic identity and status fields without extra API calls.
- `artifact_types`: query deployable artifacts and return `endpoint_supported_methods` and `supported_inference_types`.
- `active_endpoints`: query endpoints that currently reference the custom model.

## Output

Returns JSON details. Common fields include `id`, `name`, `description`, `status`, `base_model`, `customization_type`, `source`, `active_endpoints` (the endpoints currently referencing the model), `artifact_types` (available artifact types), `create_time`, and `update_time`.

**Note:** Before deletion, check `--transform active_endpoints`. A non-empty result means that deleting the model will break an existing inference path.
