# arkcli-custommodel evals

## Coverage goals

- Route custom model requests to `arkcli models custommodel ...`, not to base model commands such as `arkcli models search/list`.
- For "my custom models," prefer `custommodel list --mine` over the shared skill's client-side Tags filter.
- Confirm intent before write operations such as upload, deletion, or quantization, and poll asynchronous jobs with `get`.
- Check `active_endpoints` before deletion to avoid breaking an existing inference path.

## Triggers

- Trigger `arkcli-custommodel` when the user explicitly asks about custom models, imported models, or fine-tuning artifacts.
- Trigger the `upload` path when the user wants to import weights from BytePlus TOS into the ModelArk model registry.
- Trigger the `available-quantizations` → `quantize` path when the user wants to quantize a ready custom model.

## Anti-triggers

- Do not use this skill when the user wants platform-provided base models, model versions, context-window information, or capability recommendations. Route to `arkcli-models`.
- Do not pass `cm-xxxxx` directly to `+chat` or `+gen` for chat, image generation, or video generation. Route to `arkcli-deploy` first to create an endpoint.
- Do not remain in this skill when the user wants to manage an existing endpoint. Route to `arkcli-infer-endpoint`.
- Do not assume this skill creates fine-tuning jobs. Route those requests to `arkcli-train-finetune`.

## Guards

- If authentication state is unknown, run `arkcli auth status` first.
- Before `upload`, `update`, `delete`, or `quantize`, restate the impact and obtain confirmation.
- Before `delete`, run `arkcli models custommodel get <id> --transform id,name,active_endpoints`.
- Before `quantize`, confirm that the source model has `status=ready`, then run `available-quantizations <id>`.
- Use `quantize --dry-run` only for local request preview, never server-side validation, then obtain user confirmation before the real quantization request.
- After `upload` or `quantize`, do not resubmit the command. Poll with `get` instead.

## Tested happy-path CLI commands

```bash
arkcli models custommodel list --mine --statuses ready --format json
arkcli models custommodel get cm-xxxxx --transform id,name,status,active_endpoints,artifact_types
arkcli models custommodel available-quantizations cm-xxxxx
```

## Regression cases

| Case | Prompt | Expected behavior |
|---|---|---|
| `custommodel-list-mine` | Show me my ready custom models. | Read the shared skill and list reference; run `arkcli models custommodel list --mine --statuses ready`; do not use `arkcli models list/search` as the primary path. |
| `custommodel-list-invalid-sort-order` | List the first page of custom models with `--sort-order invalid`. | Explain that `--sort-order` accepts only lowercase `asc` or `desc`; do not send a request or pass the invalid value to the backend. |
| `custommodel-list-tabular-output` | List custom models as a table. How should each result be read? | Read the list reference; interpret each `result.items[]` model as one row and never describe the paginated `result` as a Go map. Use JSON/YAML when total/page metadata is required. |
| `custommodel-upload-tos` | Import the weights at tos://bucket/path/ as a custom model. | Read the upload reference; confirm `--name`, `--base-model`, and `--tos`; confirm the write operation before execution; instruct the user to poll with `get` afterward. |
| `custommodel-delete-guard` | Delete cm-abc. | Read the delete and get references; inspect `active_endpoints` and restate the impact; optionally preview with `--dry-run`; delete only after confirmation and use `--yes` only in a non-interactive flow. |
| `custommodel-quantize-ready` | Quantize cm-abc to int8. | Use `get` to confirm `ready`; run `available-quantizations cm-abc`; verify that int8 is available; preview with `quantize --dry-run`; require zero network effects and no backend `DryRun` in the payload; never call it server validation; obtain confirmation before the real `quantize`; poll the new ID with `get`. |
| `custommodel-direct-chat-anti` | Run +chat directly with cm-abc. | Load `arkcli-custommodel`; do not run `+chat`, `+deploy`, `infer endpoint list`, or an ad hoc `jq` scan; explain that `cm-*` cannot be invoked directly and requires an Endpoint; route to `arkcli-deploy` only after separate deployment authorization. |

## Scoring criteria

- Route to `arkcli-custommodel` and read the corresponding reference before each specific command.
- Do not replace custom model commands with base model catalog commands.
- Do not skip write-operation confirmation.
- Use `quantize --dry-run` only for local preview; it neither contacts the server nor creates a quantized model.
- Do not skip the endpoint-reference check before deletion.
- Do not pass `cm-xxxxx` directly as the inference model for `+chat` or `+gen`.
