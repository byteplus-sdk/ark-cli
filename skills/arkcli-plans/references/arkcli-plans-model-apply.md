# plans model-apply

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

Set the **ark-code-latest route target** of a Coding Plan: `auto` (smart scheduling — the platform picks the underlying model) or a concrete shadow model from the plan's model list. Uses the same backend Actions as the console's route-model confirmation, takes effect immediately, and stays in sync with the console. Write operation.

## Commands

BytePlus requires an explicit plan target; this command has no default plan.

```bash
# Interactive terminal: omit --model to open the route-target selector
# (the currently active target is tagged [active] and pre-highlighted)
arkcli plans model-apply --plan coding-plan

# Non-interactive / agent: --model is required (model id, output name, or "auto")
arkcli plans model-apply --plan coding-plan --model auto
arkcli plans model-apply --plan coding-plan-team --model doubao-seed-code-251028
```

## Parameters

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `--plan` | Yes | string | `coding-plan` / `coding-plan-team` |
| `--model` | Required when non-interactive | string | Route target: a model `model_id`, an `output_name`, or `auto`; may be omitted on an interactive terminal to open the selector |

## Response

```json
{
  "plan": "coding-plan",
  "previous_selected_model_id": "doubao-seed-code-251028",
  "applied": {
    "model_id": "auto",
    "output_name": "auto"
  }
}
```

| Field | Description |
|-------|-------------|
| `plan` | Echoes the input |
| `previous_selected_model_id` | The shadow model id that ark-code-latest resolved to before this write (empty when unset) |
| `applied.model_id` / `applied.output_name` | The route target just written; both fields are `auto` for smart scheduling |

## Notes

- **Valid targets come from the [`plans model-list`](arkcli-plans-model-list.md) output**; an out-of-list `--model` fails with `does not support route target` plus the available options
- Team plans (`coding-plan-team`) are scoped to the **current identity's seat**: the command errors when the account holds no active seat — subscribe a seat in the console first
- After writing, the CLI re-reads the model list to verify; while the control plane is still syncing, a mismatch only prints a stderr `warn` and does not change the exit code — double-check with `plans model-list`
- This command only changes the route target; it does not touch local agent configuration. To switch the model used by local agents, see `helper configure --model` in [`../../arkcli-helper/SKILL.md`](../../arkcli-helper/SKILL.md)

## See Also

- [arkcli-plans](../SKILL.md) — skill overview
- [`plans model-list`](arkcli-plans-model-list.md) — list valid route targets and the active one
- [arkcli-shared](../../arkcli-shared/SKILL.md)
