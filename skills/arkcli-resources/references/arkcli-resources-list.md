# resources list detailed reference

> **Prerequisite**: Read [`../SKILL.md`](../SKILL.md) first.

## Flag overview

| Parameter | Required | Default | Description |
|------|------|------|------|
| `--profile` | No | active | Explicit target profile; uses `RebuildForProfile` after the P0-A correction to switch identity for the control-plane request |
| `--modality` | No | `text` | `text` / `image` / `video` |

## Dispatch logic

```
profile.Type:
  platform   → ListEndpoints (PageAll) lists all active endpoints by modality
               filtering (Endpoint.ModelReference.FoundationModel.name
               → GetFoundationModel → task_types/filter_task_types;
               unknown stays visible with modality_confidence=unknown and a stderr warning)
  coding-plan
    text     → ListArkCodeLatestModel (AccountID is required and derived from SSO)
    image    → uses platform ListEndpoints + modality filter (S10, commit f69be53)
    video    → same as image
```

## Endpoint modality contract

The platform path uses this control-plane relationship:

```text
Endpoint ID
  -> ModelReference.FoundationModel.name
  -> GetFoundationModel
  -> task_types / filter_task_types
  -> text | image | video | unknown
```

- Endpoint names, FoundationModel names, and display names are identifiers, not capability evidence.
- Dola, Dreamina, Seedream, Seedance, and future branding may appear in examples, but must never be used as image/video routing rules.
- Missing, unknown, or conflicting task metadata produces `modality_confidence="unknown"`. The item remains visible with a warning for review, but selecting it as a modality default or generating through it still requires authoritative metadata or an explicit `--modality`.

## Resolve an Endpoint

When the user supplies an Endpoint and needs its binding, workflow, modality,
or Region, run:

```bash
arkcli resources resolve <endpoint-id> --format json
```

Interpret the binding fields before the capability fields:

- `endpoint_model_type="CustomModel"`: `model_id` and `custom_model_id` are
  the same bound `cm-...` ID. `base_model_*` is lineage and the authoritative
  source for task, API, and modality capabilities; it is not the Endpoint's
  bound identity.
- `endpoint_model_type="FoundationModel"`: `model_id` retains its existing
  foundation-model meaning and `custom_model_id` is absent.
- If Custom Model lineage metadata cannot be loaded, preserve the known
  `model_id == custom_model_id`, keep capabilities unknown, and inspect
  `resolution_warnings`. Never substitute `base_model_id` or guess from names.

Use `supported_workflows`, `generation_modality`, and `requires_user_intent`
only after the binding has been interpreted. `resource_region` is the
authoritative Endpoint Region for a stateless Endpoint plus API key call.

## Output format

```json
{
  "profile": "<name>",
  "type": "<platform | coding-plan>",
  "modality": "<text|image|video>",
  "items": [
    {"id": "<id>"},
    {"id": "<id>", "is_default": true}
  ],
  "current_default": "<id-or-empty>",
  "item_count": <n>
}
```

`is_default` appears only when items[].id matches `Resources.<modality>.Default` in profile.yaml.

## Differences from the old list format (before 0.1.16)

Version 0.1.16 removed all static `profile.AvailableTextModels` / `AvailableImageModels` / `AvailableVideoModels` lists (S2). All data is now fetched in real time through `resources list`. This means:

- Logic in old scripts that reads `profile.yaml.available_text_models` no longer works.
- When an Agent needs the "available model list", it MUST run `resources list` and cannot assume that profile.yaml contains it.
- profile.yaml now stores only defaults (`Resources.<modality>.Default`), not available lists.

## Difference from `arkcli api ListEndpoints`

| Dimension | `arkcli resources list` | `arkcli api ListEndpoints` |
|------|------------------------|----------------------------|
| Entry point | Product command (shortcuts) | Raw API explorer |
| Dispatch | Routes endpoint / plan / coding by profile.Type | Always ListEndpoints |
| Output | Minimal `[{"id": ...}, ...]` | All Endpoint fields |
| Identity scope | `RebuildForProfile` switches to the target identity | Uses the active profile |

Agents should prefer `arkcli resources list` and fall back to `arkcli api ListEndpoints --params ...` only when full Endpoint fields (status / quota / created_at) are required.
