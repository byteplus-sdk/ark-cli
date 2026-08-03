---
name: arkcli-resources
version: 1.1.0
description: "arkcli resources real-time control-plane queries: list resources and resolve an Endpoint into its authoritative bound model identity, base-model lineage, capabilities, workflows, and Region. Read-only; does not write profile.yaml."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli resources --help"
---

# arkcli resources

**CRITICAL — Before starting, you MUST use the Read tool to read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md), which contains authentication gates, configuration troubleshooting, and command selection order.**

## Usage principles

- The `arkcli resources` domain is strictly read-only. Even when a create intent appears in the context of a resources list, first load [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md); this Skill must not create, update, or delete resources.
- Keep an ordinary product intent such as "deploy this model and give me an Endpoint" on `arkcli +deploy`. That online workflow does not support Client Preview: **you must not run `arkcli +deploy --dry-run`**, and you must not silently downgrade the product workflow to raw CRUD just to obtain a preview.
- After handing off to the Deploy Skill, perform only read-only checks, then restate the model, name, Region, configuration, and billing impact. Until the user gives a fresh explicit confirmation in the current turn, **you must not execute the real** `arkcli +deploy`. Never add `--yes`, pipe `echo Y`, or set `ARKCLI_ALLOW_HEADLESS_ACTIVATION` on the user's behalf.
- Only when the user explicitly asks for raw CRUD, an exact CreateEndpoint request, or a CI/script preview should you route to [`../arkcli-infer-endpoint/SKILL.md`](../arkcli-infer-endpoint/SKILL.md) and use the leaf command `arkcli infer endpoint create ... --dry-run`. A real execution still requires a new confirmation after the preview.
- `arkcli resources list` is a read-only, real-time control-plane query. It calls the upstream service every time and has no local cache.
- `arkcli resources resolve <ep-id>` checks `endpoint_model_type` before interpreting model references. For a Custom Model Endpoint, `model_id` and `custom_model_id` remain the bound `cm-...` identity; `base_model_*` is lineage and capability metadata only.
- Dispatch follows profile.Type: platform → `ListEndpoints`; coding-plan → the corresponding plan API.
- **The team edition `coding-plan-team` behaves the same as the corresponding personal edition**: coding-plan-team ≈ coding-plan (it includes text-generation models, while `--modality image|video` uses the platform endpoint pool). Resource dispatch, base_url, and model lists are identical to the personal edition; only the credentials come from a team seat.
- This skill does not change defaults. When the user wants to change a default, use `profile set-default` from [`../arkcli-profile/SKILL.md`](../arkcli-profile/SKILL.md).
- `--profile X` genuinely switches identity (P0-A correction): the control-plane request uses X's token / UserID, rather than making the request as active=A and merely displaying B's resources.

## Applicable scenarios

- The user asks which endpoints / models are available under the current profile.
- The user gets `<id> is not in the available list` when running `profile set-default`; return here to view the actual available IDs.
- The user gets `InvalidEndpointOrModel.NotFound` when running `+chat / +gen --model`; return here to confirm that the ID is visible under the active profile.
- The user switched profiles and wants to know how the resources available under the new profile differ from the old profile.
- The user supplies an `ep-...` ID and needs its authoritative model binding, workflow, generation modality, or Region.

## Anti-trigger signals

- The user wants to **find a model** / **choose a model** / asks "which model is strongest / offers the best value" → route to [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md) (with enrichment + weighted ranking).
- The user wants to **create an endpoint** → route to [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md) (`arkcli +deploy`); only inspect, restate, and request confirmation in this turn.
- The user explicitly asks for a **raw CreateEndpoint / CI / exact request preview** → route to [`../arkcli-infer-endpoint/SKILL.md`](../arkcli-infer-endpoint/SKILL.md) and use `infer endpoint create --dry-run`.
- The user wants to **manage an endpoint** (start / stop / get / update / detailed list) → route to [`../arkcli-infer-endpoint/SKILL.md`](../arkcli-infer-endpoint/SKILL.md).

## Difference between resources and models

| Dimension | `arkcli resources list` | `arkcli models ...` |
|------|------------------------|----------------------|
| Scope | "What can I use" under the current profile | Full-platform foundation model catalog |
| Output | Endpoint ID (`ep-xxx`) or plan model name | All foundation_model fields + ArkModels enrichment |
| Dispatch | Switches endpoint / plan / coding APIs by profile.type | General ListFoundationModel |
| Cache | None | Cache scoped by profile/region/project |
| Primary purpose | Set defaults and verify whether `--model <id>` is active | Find models, compare models, and verify capabilities |

In short, `resources list` answers **"What can I (the current profile) use?"**, while `models` answers **"What is available on the platform?"**

## Agent quick execution order

1. If the user supplies an `ep-...` ID → `arkcli resources resolve <ep-id> --format json`; inspect `endpoint_model_type` and `model_id` before workflow fields.
2. If the current profile is uncertain → `arkcli profile show --format json` (inspect `type`).
3. Text resources → `arkcli resources list --modality text --format json`.
4. Image / video resources → `arkcli resources list --modality image --format json` / `--modality video`.
5. Compare multiple profiles → run `--profile A --modality text` and `--profile B --modality text` separately.
6. In the output, `is_default: true` marks the current profile's default. To switch the default, use `arkcli profile set-default`.

## Command overview

| Command | Description |
|------|------|
| `arkcli resources list` | List available resource IDs for the specified modality under the current or specified profile (real-time). |
| `arkcli resources resolve <endpoint-id>` | Resolve the Endpoint's authoritative bound identity, base-model lineage, candidate workflows, generation modality, and Region. |

## Output format

```json
{
  "profile": "platform_ap-southeast-1_default",
  "type": "platform",
  "modality": "text",
  "items": [
    {"id": "ep-20260424-aaaaa"},
    {"id": "ep-20260424-bbbbb", "is_default": true},
    {"id": "ep-20260424-ccccc"}
  ],
  "current_default": "ep-20260424-bbbbb",
  "item_count": 3
}
```

`is_default` appears only when `items[].id == current_default`. If the default is an empty string, no item has `is_default`.

## Endpoint resolution contract

- `endpoint_model_type="CustomModel"`: `model_id == custom_model_id == "cm-..."`. Read `base_model_*` only as lineage and the source of capability metadata; never replace the actual binding with `base_model_id`.
- `endpoint_model_type="FoundationModel"`: `model_id` keeps the existing foundation-model meaning.
- Read `supported_workflows`, `generation_modality`, and `requires_user_intent` to route the next task.
- A non-empty `resolution_warnings` is fail-soft output. Do not guess a binding from ambiguous Custom Model and Foundation Model references.

## Common errors

- Under a coding-plan profile, `resources list --modality image|video` no longer fails fast (S10): it uses the platform control-plane `ListEndpoints` to list endpoint IDs already created with `+deploy` under the same account. The user can pass them to `+gen --model <ep-id>` or `profile set-default --modality image <ep-id>`. An empty list means the user has not deployed an endpoint on platform; first run `arkcli +deploy <model>`.
- `coding-plan resources list: missing AccountID (run arkcli auth login first)` → Only the text path requires AccountID. SSO is not logged in, or claims.Sub is empty while parsing the token; run `arkcli auth login` again.
- `ListEndpoints: NotLogin / Unauthorized` → Login state/STS expired, or profile X supplied through `--profile X` has no token in the identity store; run `auth login` first.
- `unsupported profile type "X" for resources list` → profile.yaml was manually changed to an unrecognized type. Use `profile show` to inspect the `type` field, and recreate it with `profile create` if necessary.

## References

- [`../arkcli-profile/SKILL.md`](../arkcli-profile/SKILL.md) — Use this after reviewing resources when you need to change a default.
- [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md) — Use this to find models or compare capabilities.
- [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md) — Enter the creation workflow when the desired endpoint is not listed.
