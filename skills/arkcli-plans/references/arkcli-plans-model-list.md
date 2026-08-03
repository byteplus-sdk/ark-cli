# plans model-list

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

List all models supported by a specified plan and identify the **currently selected ark-latest-model** (the actual model ID aliased to `<latest-name>`). Read operation.

## Commands

BytePlus requires an explicit plan target; this command has no default plan.

```bash
# View Coding Plan personal-edition models
arkcli plans model-list --plan coding-plan

# View Coding Plan team-edition models
arkcli plans model-list --plan coding-plan-team
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `--plan` | Yes | string | Explicitly choose `coding-plan` or `coding-plan-team` |

## Return value

```json
{
  "plan": "coding-plan-team",
  "ark_latest_model_id": "seed-2-0-pro-260215",
  "models": [
    {
      "model_id": "seed-2-0-pro-260215",
      "is_ark_latest": true
    },
    {
      "model_id": "seed-pro-32k-260215",
      "is_ark_latest": false
    }
  ]
}
```

| Field | Description |
|------|------|
| `plan` | Echo of the input argument |
| `ark_latest_model_id` | Specific model ID currently aliased to `ark-latest-model`; personal edition uses the Enabled field from subscription-level `ListArkCodeLatestModel`; team edition reads `SeatInfo.ExtraConfig.ArkCodeLatestMappingModelID` from the user's seat |
| `models[].is_ark_latest` | Whether this model is the current target of ark-latest |

## Notes

- Personal and team editions use different ark-latest data sources:
  - Personal: the server returns the model list + Enabled flag at subscription granularity.
  - Team: reads `ExtraConfig.ArkCodeLatestMappingModelID` from the user's seat.
- When there is no subscription / seat, returns an empty array + empty `ark_latest_model_id` and **does not report an error**.
- The data represents models allowed during the subscription period and is not equivalent to all globally callable models. For trial models, use [`../arkcli-models/`](../../arkcli-models/SKILL.md).

## References

- [arkcli-plans](../SKILL.md) -- Skill overview
- [`plans get`](arkcli-plans-get.md) -- First view which plans you hold
- [arkcli-models](../../arkcli-models/SKILL.md) -- Model queries not limited to a plan
- [arkcli-shared](../../arkcli-shared/SKILL.md)
