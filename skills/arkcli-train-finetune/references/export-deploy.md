# Select, Export, and Deploy Fine-Tuning Artifacts

This workflow supports two entry points:

- The user provides an `mcj-...` and asks to select, export, and deploy artifacts based on training performance.
- The user provides one or more `cm-...` resources and asks which inference types they support and how to deploy them.

In addition to the shared skill, read the following skills when entering the corresponding stage:

- Query custom models, quantize them, and prepare deployable versions: [`../../arkcli-custommodel/SKILL.md`](../../arkcli-custommodel/SKILL.md).
- Create an online inference endpoint: [`../../arkcli-deploy/SKILL.md`](../../arkcli-deploy/SKILL.md).
- Configure or subsequently manage an endpoint at a lower level: [`../../arkcli-infer-endpoint/SKILL.md`](../../arkcli-infer-endpoint/SKILL.md).

Do not copy commands from memory. At each stage, read the references required by the corresponding skill and follow its write-operation, billing, and confirmation rules.

## Command summary

| Command | When to use | Common parameters |
|---|---|---|
| `arkcli train finetune metrics <mcj-id>` | Select the best-performing step | `--metric`, `--from-step`, `--to-step`, `--output` |
| `arkcli train finetune artifacts list <mcj-id>` | List artifacts | Use `--format table` for human selection or `--lite` to skip secondary custom-model name queries |
| `arkcli train finetune artifacts export <mcj-id>` | Export artifacts as `cm-*` custom models | Repeat `--artifact-name`; optionally use `--custom-model-name-prefix` |
| `arkcli models custommodel get <cm-id>` | Retrieve custom model details | `--transform artifact_types,active_endpoints` |
| `arkcli models custommodel available-quantizations <cm-id>` | Check whether a new deployable version can be created | Always run before quantization |
| `arkcli models custommodel quantize <cm-id>` | Create a new quantized version | Use `--quantization`; asynchronously returns a new `cm-*` |
| `arkcli infer endpoint capability get --custom-model-id <cm-id>` | Query supported inference types | Run before deployment |
| `arkcli +deploy` | Create or reuse an endpoint | It does not support `--dry-run`; restate final inputs and execute only after confirmation |

## A. Select the Best-Performing Step from an MCJ

If the user directly provides a `cm-...`, skip to section C.

### 1. Retrieve the job and available metrics

```bash
arkcli train finetune get <mcj-id>
arkcli train finetune metrics <mcj-id>
```

Use `get` to confirm the fine-tuning type, training method, and job phase. If the job has not produced valid metrics or artifacts, explain its current phase and stop the export flow.

For a job that uses full fine-tuning, stop the CLI export-and-deploy flow and explain that ArkCLI currently cannot deploy artifacts produced by full fine-tuning. The user must complete deployment in the console. Do not continue with `artifacts export`, custom model deployment, or `+deploy`.

On the first `metrics` call, omit the metric name and obtain exact names from `available_metrics`. For a large result, write it to a file with `--output`, then calculate with a structured JSON tool rather than scanning a long series manually.
When restricting the range with both `--from-step` and `--to-step`, `to-step` must be strictly greater than `from-step`.

### 2. Select the performance metric used for ranking

Apply the following priority. Every selected name must come from `available_metrics`.

| Job type | First choice | Fallback |
|---|---|---|
| Non-RL | Minimize a loss metric in the test, eval, or validation scope | Minimize a loss metric in the train scope |
| RL | Maximize `final_rewards.mean` in the test, eval, or validation scope | Maximize `final_rewards.mean` in the train scope |

Matching rules:

- Treat `test`, `eval`, and `validation` as evaluation scopes with equal priority. If multiple candidates exist, show their names and state which one is used.
- For RL, recognize the `final_rewards` metric family, including names such as `final_rewards`, `final_rewards.mean`, `final_rewards.best@N`, and `final_rewards.worst@N`.
- Prefer `final_rewards.mean`, then an exact `final_rewards`, because they represent aggregate rollout performance. Treat `best@N` and `worst@N` as supporting metrics; do not use either to rank artifacts unless the user explicitly chooses it.
- Do not silently substitute accuracy, a generic reward, gradient norm, or another metric for the primary metric above.
- If neither the preferred nor fallback metric exists, list the actual available metrics and ask the user to choose. Do not guess.

Record:

- The exact metric name and why it was selected.
- The best metric value.
- The corresponding step or steps.
- Whether the metric belongs to an evaluation set or the training set.

Retain every step tied at the best value. If the user requests top-k, sort in the correct metric direction and record the top-k steps. Do not treat neighboring steps as multiple "best-performing steps" by default.

## B. Map the Best-Performing Step to an Artifact and Export It

### 1. List artifacts

```bash
arkcli train finetune artifacts list <mcj-id>
```

Extract the step from the artifact name or structured fields. Automatically map only artifacts whose steps can be parsed reliably. If the step cannot be parsed, show the artifact list and ask the user to choose.

For each best-performing step:

1. Prefer an artifact with an exact step match.
2. If no exact match exists, calculate the absolute distance to each exportable artifact step.
3. Select the nearest artifact. If multiple artifacts tie at the same distance, retain every tied candidate.
4. Show `metric step → artifact step → distance` explicitly. Never describe a nearest artifact as an exact match.

Deduplicate artifacts mapped from different best-performing steps to avoid exporting the same artifact more than once.

### 2. Obtain user confirmation

Before export, display:

- MCJ ID, fine-tuning type, and training method.
- Selected performance metric, best value, and best-performing step.
- Recommended artifact and mapping distance.
- Current artifact export status.
- Existing custom model ID, if already exported.

Ask the user to confirm the final artifact or artifacts to export. Allow the user to add or remove candidates.

### 3. Export

Run the following only for user-confirmed artifacts that have not already been exported:

```bash
arkcli train finetune artifacts export <mcj-id> \
  --artifact-name <artifact-1> \
  --artifact-name <artifact-2>
```

If a naming prefix is needed, confirm it first and use only a parameter supported by the current `--help`. Reuse the existing `cm-...` for an already exported artifact; do not export it again.

Explain that export registers an MCJ artifact as a custom model in the model registry; it does not download model weights. Record the custom model ID and status corresponding to each artifact.

Common export statuses:

- `Available`: available for export.
- `Exported`: already exported; reuse `custom_model_id`.
- `Exporting`: export is in progress; query again later.
- `ExportFailed` / `Expired`: not directly deployable; show the status and ask the user how to proceed.

## C. Query Custom Models and Deployable Versions

For every user-provided or newly exported `cm-...`, read and follow the `arkcli-custommodel` skill.

At minimum, run:

```bash
arkcli models custommodel get <cm-id>
arkcli models custommodel get <cm-id> --transform artifact_types,active_endpoints
arkcli infer endpoint capability get --custom-model-id <cm-id>
```

Summarize:

- Custom model name, source, status, and base model.
- Existing endpoints.
- Inference types directly supported by the current model.
- Additional inference types that may become available after creating a new version through quantization or another supported method.

If the target inference type is not currently supported, follow the `arkcli-custommodel` skill:

1. Query `available-quantizations` and the inference types each quantization method is expected to support.
2. Explain the potential impact of quantization on accuracy, performance, and resource options.
3. After user confirmation, create a new quantized `cm-...`.
   - `custommodel quantize --dry-run` is a command-local Client Preview and emits `preview.v1`; it never sends the quantization request.
4. Poll the new model until it reaches `ready`.
5. Query the new `cm-...` again for details and its actual supported inference types.

Do not create a new version merely because a quantization option exists. The source and quantized models are different resources; preserve a clear mapping between them.

## D. Select and Deploy

Show a deployment-candidate table containing at least:

- Custom model ID and name.
- Source artifact and step, when known.
- Corresponding best metric and value, when known.
- Whether it is the original or quantized version.
- Currently supported inference types.
- Existing running endpoint, if any.

Ask the user:

1. Which `cm-...` resource or resources to deploy.
2. Which supported inference type to use for each model.
3. The endpoint name and configuration required by that inference type.

After the user chooses, read and follow the `arkcli-deploy` skill:

1. Restate the final inputs and impact for each target.
2. Show the endpoint to be created or reused, the selected inference type, and billing impact.
3. Obtain final confirmation, then perform the deployment.
4. Record every `cm-... → ep-...` mapping and endpoint status.

Deploying multiple models creates multiple billable resources. Make the number of targets explicit. Never treat confirmation of artifact export as confirmation of endpoint deployment.
