# plans model-list

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

List two model sets for a specified plan: `models` contains route targets behind
`ark-code-latest`, while `direct_models` contains ModelNames that can be placed
directly in an inference request or Agent configuration. Both sets may include
load and capability metadata. Read operation.

## Commands

When the current identity holds exactly one active Coding Plan, omitting
`--plan` auto-detects that plan. If both personal and team editions are active,
the CLI asks for an explicit target instead of guessing.

```bash
# Auto-detect the only active Coding Plan
arkcli plans model-list

# View Coding Plan personal-edition models
arkcli plans model-list --plan coding-plan

# View Coding Plan team-edition models
arkcli plans model-list --plan coding-plan-team

# Query only directly invocable ModelNames (VirtualModelID only)
arkcli plans model-list --plan coding-plan --mode direct

# Query only ark-code-latest smart-routing candidates (LatestModel only)
arkcli plans model-list --plan coding-plan --mode routing
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `--plan` | No | string | Choose `coding-plan` or `coding-plan-team`; when omitted, auto-detect the only active Coding Plan and require an explicit value if multiple plans are active |
| `--mode` | No | string | `all`, `direct`, or `routing`; defaults to `all`. `direct` fetches only direct ModelNames, while `routing` fetches only smart-routing candidates |

## Return value

```json
{
  "plan": "coding-plan-team",
  "edition": "enterprise",
  "tier": "pro",
  "selected_model_id": "seed-2-0-pro-260215",
  "models": [
    {
      "model_id": "seed-2-0-pro-260215",
      "model_name": "Seed 2.0 Pro",
      "output_name": "seed-2-0-pro",
      "context_window": 262144,
      "max_input_tokens": 229376,
      "max_completion_tokens": 65536,
      "input_modalities": ["text", "image", "video"],
      "load": {"status": 1, "level": 2, "score": 0.46, "coverage": 0.92, "updated_at": 1789999999},
      "selected": true
    },
    {
      "model_id": "seed-pro-32k-260215",
      "model_name": "Seed Pro 32K",
      "output_name": "seed-pro-32k"
    }
  ],
  "direct_models": [
    {
      "model_name": "seed-2.0-pro",
      "context_window": 262144,
      "max_input_tokens": 229376,
      "max_completion_tokens": 65536,
      "input_modalities": ["text", "image", "video"],
      "load": {"status": 1, "level": 1}
    }
  ]
}
```

| Field | Description |
|------|------|
| `plan` | Echo of the input argument |
| `edition` | Plan API edition: `personal` for a personal plan or `enterprise` for a team plan |
| `tier` | The active plan tier sent as `BizInfo` |
| `selected_model_id` | Current backend model ID selected for the route; personal edition uses the subscription-level `Enabled` field, while team edition reads the user's seat mapping; omitted when unset |
| `models[]` | Route candidates behind `ark-code-latest`; use these with `plans model-apply`, not as the direct ModelName catalog |
| `models[].model_id` | Backend route-target ID for control-plane writes |
| `models[].model_name` | Human-readable display name; do not use it as the invocation ID |
| `models[].output_name` | Companion output name for route writes; do not infer direct invocation support from it |
| `models[].selected` | `true` when this item is the current route target; omitted when false |
| `direct_models[]` | Models directly invocable by the current plan, sourced from the VirtualModelID API |
| `direct_models[].model_name` | Exact ModelName to put in an inference request or Agent configuration |
| `*.context_window` | Context-window size in tokens; omitted when unavailable |
| `*.max_input_tokens` | Maximum input-token length; omitted when unavailable |
| `*.max_completion_tokens` | Maximum output-token length; omitted when unavailable |
| `*.input_modalities` | Supported input modalities such as `text`, `image`, and `video` |
| `*.load` | Server load snapshot; only `status=1` with `level=1/2/3` means low/medium/high load respectively |

## Notes

- With an explicit `--plan`, the CLI resolves that target's active tier. Without it, the CLI queries the current identity's Coding Plan subscriptions and selects only when exactly one is active; multiple active plans or any incomplete status lookup require an explicit target. Every model-list request still carries `Edition` and `BizInfo`.
- `--mode all` queries both model interfaces. `--mode direct` queries only VirtualModelID and returns an empty `models` array. `--mode routing` queries only LatestModel and returns an empty `direct_models` array. Team routing may still read seat information to identify the selected route.
- Personal and team editions use different sources for the current ark-code-latest selection:
  - Personal: the server returns the model list + Enabled flag at subscription granularity.
  - Team: reads `ExtraConfig.ArkCodeLatestMappingModelID` from the user's seat.
- Do not mix the two lists: use `models[]` only for changing the `ark-code-latest` route, and use `direct_models[].model_name` for direct invocation or Agent configuration.
- Model capability fields come directly from the plan model interface's `ModelCapability`. Missing fields are omitted; do not infer them from model names or overwrite them with another catalog.
- The data represents models allowed during the subscription period and is not equivalent to all globally callable models. For trial models, use [`../arkcli-models/`](../../arkcli-models/SKILL.md).

## References

- [arkcli-plans](../SKILL.md) -- Skill overview
- [`plans get`](arkcli-plans-get.md) -- First view which plans you hold
- [arkcli-models](../../arkcli-models/SKILL.md) -- Model queries not limited to a plan
- [arkcli-shared](../../arkcli-shared/SKILL.md)
