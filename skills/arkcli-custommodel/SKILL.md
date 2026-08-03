---
name: arkcli-custommodel
version: 0.1.0
description: "Manage BytePlus ModelArk custom models identified by `cm-*`: list, inspect, upload, update, delete, quantize, and prepare deployment. Also use this skill whenever a user asks to chat, invoke, or test inference directly with a `cm-*` ID, so the Endpoint boundary is enforced before any deployment or inference action."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli models custommodel --help"
---

# arkcli models custommodel

> **BytePlus capability status: pending audit.** This English file is a baseline translation, not proof that the capability is available in BP. Check the capability matrix before execution, and do not perform write operations until support, region, permissions, and dependencies have been confirmed.

**CRITICAL — Before starting, MUST use the Read tool to read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md), which defines authentication gates, configuration troubleshooting, and shared safety rules.**
**CRITICAL — Before running any `models custommodel` command, use the Read tool to read the corresponding reference document. Never invoke a command without reading its reference first.**
**CRITICAL — Confirm user intent before any write operation (`upload`, `update`, `delete`, or `quantize`). Before deletion, confirm whether any endpoint still references the model.**
**CRITICAL — This skill targets BytePlus. Before running control-plane commands, confirm that the active profile, project, and region all belong to the intended BytePlus environment.**

## Guard Checklist and Operating Principles

- Prefer `arkcli models custommodel ...` for custom model requests.
- Although these are standard CLI-style commands, their implementation entry point remains under `shortcuts/models/`.
- Fall back to [`../arkcli-api-explorer/SKILL.md`](../arkcli-api-explorer/SKILL.md) only when no product command covers the request.
- Do not use this skill to query the base model catalog. Route those requests to [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md).
- For write operations and asynchronous jobs, explain the impact, polling method, and next steps. Do not stop after issuing a single command.

## When To Trigger and Supported Scenarios

- Import trained or fine-tuned weights from BytePlus Torch Object Storage (TOS) into the ModelArk model registry.
- List custom models in the current account, including requests such as "Which custom models do I own?" or "What are their statuses?"
- Inspect custom model details, artifact types, and active endpoint references.
- Update a custom model's display name or description.
- Delete an unused custom model.
- Quantize a `ready` custom model to prepare it for `+deploy`.

## When NOT To Trigger

- Find platform-provided base models → use `search`, `list`, or `get` from [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md).
- Invoke a custom model directly for inference → first use `+deploy`, then use `+chat` or `+gen`.
- Create a fine-tuning job itself → route to [`../arkcli-train-finetune/SKILL.md`](../arkcli-train-finetune/SKILL.md).
- **Register a fine-tuning step (`global_step_N`) as a `cm-` resource** (that is, export a training artifact) → use `arkcli train finetune artifacts list / export` from [`../arkcli-train-finetune/SKILL.md`](../arkcli-train-finetune/SKILL.md). **Do not** use this skill's `upload`: `upload` is for a user's own TOS files and calls the `UploadModel` Action, whereas MCJ outputs use `CreateCustomModel`. These are different APIs.
- Manage an existing endpoint after obtaining its endpoint ID → route to [`../arkcli-infer-endpoint/SKILL.md`](../arkcli-infer-endpoint/SKILL.md).

## Direct inference boundary for `cm-*`

- A request to chat, invoke, or test inference directly with a `cm-*` ID must still load this skill. Explain that `cm-*` identifies a custom-model resource and cannot be passed directly to `+chat` or `+gen`; inference requires a separate Endpoint.
- The request alone does not authorize deployment, listing account Endpoints, or inference. Until the user separately asks to continue, do not run `arkcli +deploy`, `arkcli +chat`, `arkcli infer endpoint list`, or an ad hoc `jq` scan.
- State the boundary and offer the next step only. After explicit authorization to deploy, route to [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md) and follow its confirmation flow.

## Core concepts

- Call resources managed by `arkcli models custommodel ...` **custom models** (`CustomModel`, with IDs such as `cm-xxxxx`). They are separate from the platform-provided base model resources (`FoundationModel`) queried through `search`, `list`, or `get` in [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md).
- A custom model has one of two sources:
  - `import` — weights imported from TOS with this skill's `upload` command through the `UploadModel` API.
  - `customization` — a model produced by a fine-tuning job and exported through `train finetune artifacts export` in [`../arkcli-train-finetune/SKILL.md`](../arkcli-train-finetune/SKILL.md). This path uses the separate OpenAPI Action `CreateCustomModel` and is not interchangeable with `upload`.
- The lifecycle is `preparation → processing → ready` on success, or `failed`. Export flows also use `exporting` and `exportfailed`.
- Quantization is a separate flow. First run `available-quantizations <id>` and inspect `supported_inference_types_by_quantization` to predict the inference types supported by each quantization method. Then submit `quantize <id> --quantization <mode>`. The result is an **independent new `cm-xxxxx`**. The source model, quantized model, and deployed endpoint are three different resources; never mix their IDs.
- A custom model ID (`cm-xxxxx`) is **not** in `<name>-<primary_version>` form and cannot be passed directly to `--model` for `+chat` or `+gen`. First obtain an endpoint through `arkcli +deploy`, then invoke the resulting `ep-xxx`. If the custom model already has a running endpoint, `+deploy` reuses it.

## Quick decisions

- The user asks for "my custom models" → read [`references/arkcli-custommodel-list.md`](references/arkcli-custommodel-list.md) and run `arkcli models custommodel list --mine`.
- The user wants a fuzzy search by name, ID, or base model display name → read [`references/arkcli-custommodel-list.md`](references/arkcli-custommodel-list.md) and run `list --search <kw>`.
- The user has a `cm-xxxxx` and wants its status, details, endpoint references, or artifact types → read [`references/arkcli-custommodel-get.md`](references/arkcli-custommodel-get.md) and use `get`.
- The user wants to import weights from TOS → read [`references/arkcli-custommodel-upload.md`](references/arkcli-custommodel-upload.md). If the user has a `tos://...` URI, confirm the TOS URI, base model, and name. If no TOS URI is available, guide the user through enabling TOS and uploading the files; do not run `upload` yet.
- The user wants to rename a model or change its description → read [`references/arkcli-custommodel-update.md`](references/arkcli-custommodel-update.md), confirm the change, and use `update`.
- The user wants to delete a model → read [`references/arkcli-custommodel-delete.md`](references/arkcli-custommodel-delete.md), inspect `active_endpoints`, and delete only after confirmation.
- The user wants to quantize a model → first read [`references/arkcli-custommodel-available-quantizations.md`](references/arkcli-custommodel-available-quantizations.md), then read [`references/arkcli-custommodel-quantize.md`](references/arkcli-custommodel-quantize.md).

## List parameter and output contracts

- `custommodel list --sort-order` accepts only lowercase `asc` or `desc`. Treat any other value as a local parameter error; do not call the API or blindly retry with another spelling.
- With `--format table` or `--format csv`, each `result.items[]` model is one row. Do not interpret the paginated response root or the `result` map as a model record. Use JSON/YAML when complete pagination metadata is required.

## Agent execution order

1. If authentication state is uncertain, run `arkcli auth status` first.
2. For "my custom models," run `custommodel list --mine` directly. **Do not apply the shared skill's default Tags filter** because the custom model service supports `--mine` natively.
3. Before upload, require all three fields: `--name`, `--base-model <foundation-model-id>`, and `--tos tos://<bucket>/<prefix>`. The server rejects requests missing any of them.
4. `upload` and `quantize` are asynchronous. After submission, poll with `custommodel get <id>`; **do not repeatedly submit `upload` or `quantize`**.
5. Before `quantize`, always run `available-quantizations <id>`. Supported quantization methods vary by base model, and the server rejects unsupported values. If the user needs pay-as-you-go by token usage or Model Unit inference, inspect `supported_inference_types_by_quantization` first.
6. `quantize --dry-run` emits local `preview.v1` and never calls `CreateQuantizedCustomModel`. Its `steps[].payload` describes only real request fields and must not contain a backend `DryRun` field. It is not server validation. Obtain explicit user confirmation before the real quantization request.
7. `delete`, `update`, and `quantize` are write operations. Restate their impact before execution.
8. `delete` prompts for [Y/N] confirmation by default. `--yes` skips the local confirmation, while `--dry-run` previews without deleting. Add `--yes` only after the user has explicitly confirmed the deletion target and impact.
9. `get --transform` is a field allowlist specific to `custommodel get`, not the global GJSON transform expression. Do not use it for nested-path queries.

## Common workflows

### 1. Import a new custom model from TOS

```
auth status → custommodel upload --name X --base-model <fm-id> --tos tos://b/p
            → custommodel get <id>  (poll until status=ready)
            → custommodel get <id> --transform 'artifact_types'  (inspect artifact types)
```

### 2. Quantize a ready custom model

```
custommodel get <id>  (confirm status=ready)
        → custommodel available-quantizations <id>  (inspect supported modes)
        → custommodel quantize <id> --quantization <mode> --dry-run
        → obtain user confirmation for the quantization target and impact
        → custommodel quantize <id> --quantization <mode>
        → custommodel get <new-id>  (the quantized result is a new cm-xxxxx; poll again)
```

### 3. Prepare a target for `+deploy`

```
custommodel list --mine --statuses ready  → select a target cm-xxxxx
        → +deploy --model cm-xxxxx ...   (reuse a running endpoint if one exists; see ../arkcli-deploy/SKILL.md)
```

### 4. Remove an unused custom model

```
custommodel get <id> --transform 'active_endpoints'  (confirm that no endpoint references it)
        → custommodel delete <id> --dry-run
        → custommodel delete <id>  (interactive confirmation) or custommodel delete <id> --yes
```

## Anti-patterns

- Do not use `arkcli models search` or `list` to find custom models. Those commands query only the `FoundationModel` catalog. Use `custommodel list --search <kw>` or `--mine` for imported or fine-tuned custom models.
- Do not run `quantize` immediately after `upload`. Upload is asynchronous and progresses through `preparation → processing → ready`. Confirm `ready` with `custommodel get <id>`, then run `available-quantizations` followed by `quantize`.
- Do not pass a mode to `quantize` unless `available-quantizations` returned it. Supported methods vary by base model.
- Do not treat a successful `quantize --dry-run` response as a created or server-validated model. It is only a local request preview; run the command again without `--dry-run` after confirmation.
- Do not pass `cm-xxxxx` directly to `--model` for `+chat` or `+gen`. First obtain an endpoint (`ep-xxx`) through `+deploy`; `+deploy` may reuse a running endpoint.
- Do not add `--yes` merely for automation. Without `--yes`, the CLI prompts for [Y/N] confirmation. Use it only after the user confirms deletion of the target `cm-xxxxx` and understands the endpoint-reference risk.
- Do not apply the shared Tags client-side filter to a request for "my" models. `custommodel list --mine` is a more accurate and efficient native server-side filter.
- Do not poll status by calling `get` too frequently. Use an interval of at least 10 seconds to avoid rate limiting.

## Command summary

| Command | Description |
|---|---|
| `arkcli models custommodel list` | Paginate and filter on multiple dimensions |
| `arkcli models custommodel get <id>` | Retrieve details or poll status |
| `arkcli models custommodel upload` | Import from TOS asynchronously |
| `arkcli models custommodel update <id>` | Update the name or description |
| `arkcli models custommodel delete <id> [--yes] [--dry-run]` | Delete irreversibly; prompts by default, `--yes` skips confirmation, and `--dry-run` previews |
| `arkcli models custommodel available-quantizations <id>` | Query available quantization methods before `quantize` |
| `arkcli models custommodel quantize <id> --quantization <mode> [--dry-run]` | Quantize asynchronously; `--dry-run` previews locally without calling the backend |

## Common fallbacks

- Authentication failure → use [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).
- The profile or region appears incorrect → use [`../arkcli-config/SKILL.md`](../arkcli-config/SKILL.md).
- The user wants a base model rather than a custom model → use [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md).
- The user wants to deploy a custom model → use [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md).
- A new capability is genuinely not covered by a product command → use [`../arkcli-api-explorer/SKILL.md`](../arkcli-api-explorer/SKILL.md).

## References

- [`references/arkcli-custommodel-list.md`](references/arkcli-custommodel-list.md)
- [`references/arkcli-custommodel-get.md`](references/arkcli-custommodel-get.md)
- [`references/arkcli-custommodel-upload.md`](references/arkcli-custommodel-upload.md)
- [`references/arkcli-custommodel-update.md`](references/arkcli-custommodel-update.md)
- [`references/arkcli-custommodel-delete.md`](references/arkcli-custommodel-delete.md)
- [`references/arkcli-custommodel-available-quantizations.md`](references/arkcli-custommodel-available-quantizations.md)
- [`references/arkcli-custommodel-quantize.md`](references/arkcli-custommodel-quantize.md)
- [`references/evals.md`](references/evals.md)
- [arkcli-models](../arkcli-models/SKILL.md) — query base models; these are separate resources from custom models
- [arkcli-deploy](../arkcli-deploy/SKILL.md) — deploy a ready custom model as an endpoint
- [arkcli-shared](../arkcli-shared/SKILL.md) — authentication, global flags, and safety rules
