# models activate

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

Proactively activate services for a specified foundation model (corresponding to the `OpenModelChargeItem` Action in section "3. Model activation" of URL2, with the API display name "Activate model service").

## Difference from implicit activation by deploy / infer-create

| Entry point | Trigger method | Applicable scenario |
|------|---------|---------|
| `arkcli models activate <name>` | The user **proactively** activates it | Prepare model availability for the business in advance; try a new model; preview the local request first |
| `arkcli deploy ...` / `arkcli infer endpoint create ...` | **Passively** triggered: automatically prompts when the model has not been activated | The user's actual goal is deployment / endpoint creation, and activation is only a prerequisite |

`activate` does not first call GetModelChargeItem to check the state. It directly initiates OpenModelChargeItem. Repeated calls for an already activated model are idempotent (handled by the backend).

## Commands

```bash
# By default, activate only the base service (inference + fine-tuning), equivalent to SubServices=["base"]
arkcli models activate dola-seed-2-1-turbo

# Client Preview: show the local request without contacting the backend or entering the [Y/N] prompt
arkcli models activate dola-seed-2-1-turbo --sub-services base,fast-infer --dry-run

# Non-interactive (CI / scripting scenario; skips the [Y/N] confirmation prompt)
arkcli models activate dola-seed-2-1-turbo --yes
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `<foundation-model-name>` | Yes | string | Foundation model name to activate; query it using `arkcli models list` / `search` |
| `--sub-services` | No | string list | Sub-services to activate, comma-separated. Valid values: `base` / `context-cache` / `fast-infer`. **When omitted, the backend activates only `base` by default** |
| `--yes` | No | bool | Skip the `[Y/N]` secondary confirmation. Required in CI / scripts |

### `--sub-services` values (authoritative URL2 definition)

| Value | Meaning |
|----|------|
| `base` | Base service (inference + fine-tuning) |
| `context-cache` | Context cache |
| `fast-infer` | Low-latency inference service |

When omitted, the backend treats it as `["base"]`. The CLI first performs local validation: a non-enum value is immediately rejected, avoiding a request that the backend would reject with `InvalidParameter.SubServices`.

### Client Preview

`--dry-run` is a command-local Client Preview. It emits the locally built
`OpenModelChargeItem` request in `preview.v1`, never contacts the backend, and
never enters confirmation or polling. It does not prove that the model exists,
permissions are valid, or the server will accept the request.

## Return value

Successful response (JSON):

```json
{
  "foundation_model": "dola-seed-2-1-turbo",
  "state": "Available",
  "sub_services": ["base", "context-cache"]
}
```

Client Preview shape:

```json
{
  "schema_version": "preview.v1",
  "dry_run": true,
  "mode": "client_preview",
  "steps": [{
    "target": "OpenModelChargeItem",
    "payload": {
      "FoundationModelName": "dola-seed-2-1-turbo",
      "SubServices": ["base"]
    },
    "fidelity": "logical"
  }]
}
```

## Common errors

| Error | Cause | Handling |
|------|------|---------|
| `invalid --sub-services value "foo"` | The sub-service name is misspelled | Use `base` / `context-cache` / `fast-infer` |
| `Operation canceled` | A value other than Y was entered at the `[Y/N]` prompt | Run the command again and confirm, or add `--yes` |
| Model does not exist | `<foundation-model-name>` is wrong or the model has been taken offline | Use `arkcli models search` to find the correct name |
| `timed out waiting for model to become available` | Activation succeeded, but the model did not become Available within five seconds | Usually harmless; retry `arkcli models get <name>` later to inspect the actual state |

## Notes

- Sub-services can be activated incrementally: for example, first `--sub-services base`, then `--sub-services context-cache` or `--sub-services fast-infer`; they do not conflict.
- `--dry-run` makes no network request, activates nothing, and consumes no usage. It is a client preview, not server validation.
- Activating an already activated model is idempotent (backend contract).

## References

- [arkcli-models](../SKILL.md) — Models skill overview
- [arkcli-shared](../../arkcli-shared/SKILL.md) — Authentication and global parameters
- URL2 document section "3. Model activation" → `OpenModelChargeItem`
