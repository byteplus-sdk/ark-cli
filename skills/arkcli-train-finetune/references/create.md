# Create a Fine-Tuning Job

- Follow the sequence below. If the user provides insufficient information, query current capabilities and make recommendations.
- If the user has already specified the model, version, training data, customization type (`--type`), and either explicit hyperparameters or a default strategy, first validate the model and capability as required in section 1, then run `create --dry-run` directly to obtain a server-side preview. If the dry run fails or returns insufficient information, query the relevant command according to the error.
- Wait for user confirmation before the real `create` operation.

## 1. Query models, training options, prices, and supported inference types

| Command | When to use | Common parameters |
|---|---|---|
| `arkcli models search [keyword]` | Query candidate base models from the active BytePlus catalog; omit the keyword to inspect the catalog when the user has not named a model | `--modality`, `--strict-filter`, `--refresh-cache` |
| `arkcli models versions <model>` | Find versions of a base model; fine-tuning support shown here is informational only | None |
| `arkcli train finetune capability get` | Query the fine-tuning types and training methods supported by a base model version or the source base model of an existing custom model; treat this as authoritative | `--model`, `--version`, `--model-id` |
| `arkcli train finetune pricing` | Query token- or instance-billed fine-tuning prices | `--model`, `--model-version`, `--type`, `--billing-method` |
| `arkcli infer endpoint capability get` | Query supported inference types for a base model (`model` + `version`) or custom model (`custom-model-id`) | `--model`, `--version`, `--custom-model-id` |

CLI flags can evolve. Use this reference directly for routine cases; inspect `--help` when a command fails or a parameter is uncertain.

Run every price and capability query under the target BytePlus profile, project, and region. Even when model or training method names match those on another platform, do not reuse that platform's prices or capability results.

Select models in this fail-closed order:

1. Query `arkcli models search [keyword]` first under the active BytePlus scope; omit the keyword when the user has not named a model. Use only exact returned names and never normalize product prefixes.
2. Before showing or using a candidate, verify it with:

   ```bash
   arkcli models versions <candidate>
   arkcli train finetune capability get --model <candidate> --version <version>
   ```

   Keep it only when the exact version exists and `supported_types` contains the requested type; LoRA SFT requires `lora`.
3. If either query fails or returns no exact eligible match, report that availability could not be confirmed. Do not name a fallback model or assemble a model-dependent estimate or create command.

After a model passes the checks above, query its price and potential inference methods:

```bash
arkcli models versions <model>
arkcli train finetune capability get --model <model> --version <version>
arkcli train finetune pricing --model <model> --type <type>
# Token billing is the default. Do not replace this with generic arkcli pricing models.
# Instance pricing requires the exact version and type. Keep custom hyperparameters identical to creation.
arkcli train finetune pricing --model <model> --model-version <version> --type <type> \
  --billing-method instance --hyperparameters '{"epoch":"3"}'
arkcli infer endpoint capability get --model <model> --version <version>
```

Explain:

- Which fine-tuning types and training methods the selected version supports.
- The billing unit and price for the selected customization type.
- `token` returns token charge items. `instance` first verifies that the exact model version and
  customization type support Instance billing, then returns resource templates, flavor unit prices,
  role-size hourly prices, and the overall hourly range. Treat the range as complete only when
  `price_complete=true`; otherwise report `missing_flavor_ids` and never interpret a missing price as zero.
- The potential post-fine-tuning inference methods for the selected version and training method.
  - A query using `--model` and `--version` returns the base model's supported inference types. Treat them only as potential inference types for a future fine-tuned artifact; the artifact is not guaranteed to support exactly the same types.
  - After training, query the actual custom model with `--custom-model-id`, then continue through the deployment skill. Do not substitute `--model-id`: that flag belongs to `train finetune capability get`, not `infer endpoint capability get`.

Default fine-tuning type and training method:

- If the user does not specify a fine-tuning type through `--type`, default to SFT.
- If the user does not specify LoRA or full fine-tuning as the training method, default to LoRA.
- Therefore, when the model supports LoRA, select the LoRA-encoded `--type` by default. If LoRA is unsupported, show the available fine-tuning types and training methods and ask the user to choose.
- If the user explicitly requests full fine-tuning, use the full fine-tuning `--type` corresponding to the selected fine-tuning type. Before continuing, explain that ArkCLI currently cannot deploy artifacts produced by full fine-tuning and that deployment must be completed in the console after training.

If the user has not selected a model, show a small set of query-proven, capability-verified candidates and ask the user to choose. Do not show unverified candidates or submit a high-cost job autonomously.

## 2. Common creation parameters

Check whether current ArkCLI flags can fully express the request:

```bash
arkcli train finetune --help
arkcli train finetune create --help
```

Common parameters for `arkcli train finetune create`:

| Parameter | Description |
|---|---|
| `--name` | Job name |
| `--description` | Job description |
| `--model` + `--model-version` | Fine-tune a base model |
| `--model-id` | Continue fine-tuning an existing custom model; mutually exclusive with `--model/--model-version` |
| `--type` | Customization type, which may encode both the fine-tuning type and training method; when omitted, apply the default SFT + LoRA policy |
| `--train-file` | Local training file; repeatable |
| `--train-tos-uri` | BytePlus Torch Object Storage (TOS) URI of uploaded training data |
| `--train-dataset` + `--train-dataset-version` | Existing training dataset reference; this skill does not create datasets |
| `--validation-file` | Local validation file; repeatable |
| `--validation-tos-uri` | TOS URI of an uploaded validation set |
| `--validation-dataset-id` + `--validation-dataset-version` | Existing validation dataset reference |
| `--validation-percentage` | Split a validation set from the training set; mutually exclusive with an explicit validation set |
| `--hyperparameters` | JSON string or `@file`; pass values as strings when required by the backend |
| `--epochs`, `--lr`, `--lora-rank`, `--beta` | Legacy convenience parameters. When supported, they are merged into hyperparameters and take precedence on conflict. |
| `--max-invalid-records-ratio`, `--max-invalid-records-number` | Maximum invalid-record ratio or count for data fault tolerance |
| `--shuffle-random-seed` | Data-order control: random shuffle, no shuffle, or a fixed seed |
| `--save-model-limit` | Number of training artifacts to retain |
| `--enable-trajectory` | Enable RL trajectory logging. Query the resulting rollout trajectories with `arkcli train finetune trajectory`; this requires an SSO-authenticated profile with access to the corresponding TLS topic and does not depend on the fine-tuning SDK. |
| `--pipeline` | RL pipeline configuration file; use only when current ArkCLI can fully express the configuration |
| `--yes` | Skip CLI confirmation. Add it only after the user reconfirms or explicitly asks for immediate creation. |

## 3. Obtain and validate training data

Do not create or manage platform datasets in this skill. Accept either:

- Local training files.
- Uploaded BytePlus TOS URIs.

If the user provides no data, ask for a training-set file or an existing data reference.

### Map `dataset_schema`

First read `dataset_schema` from `arkcli models finetune-config <model> <version> --type <type>`, then interpret it using the table below. For field details, examples, and restrictions, follow the BytePlus ModelArk [Model fine-tuning dataset format description](https://docs.byteplus.com/en/docs/ModelArk/1099461).

| `dataset_schema` | Dataset format |
|---|---|
| `LLMSft` | Supervised Fine-Tuning dataset for language models |
| `LLMRL` | Reinforcement-learning dataset for language models |
| `PromptResponse` | SFT for text generation models |
| `ImageRecognitionSFT` | SFT for multimodal models; also compatible with text generation models |
| `ImageGenerationSFT` | SFT for image generation models |
| `VideoGenerationSFT` | SFT for video generation models |
| `TextDPO` | DPO for text generation models |
| `ImageRecognitionDPO` | DPO for multimodal models; also compatible with text generation models |
| `TextRL` | Reinforcement learning for text generation models |
| `ImageRecognitionRL` | Reinforcement learning for multimodal models; also compatible with text generation models |
| `Text` | Continued Pre-Training for multimodal or text generation models |

If `finetune-config` does not return `dataset_schema`, do not guess. Check the BytePlus dataset-format documentation, perform the local structural checks below, and use `arkcli train finetune create --dry-run` for model-aware server-side validation. If the format remains ambiguous, ask the user to confirm it.

### Local offline checks

For local files, use the BytePlus ModelArk [Model fine-tuning dataset format description](https://docs.byteplus.com/en/docs/ModelArk/1099461) to check:

- The file is readable, correctly encoded, and reasonably sized.
- Every non-empty JSONL line is a JSON object.
- Sample count and invalid-line count.
- Required fields, types, and content structure from the documentation.

Use a structured JSON parser rather than regular expressions to validate JSON. Report the exact sample count and failing line numbers. Never describe a local check as authoritative platform validation.

Token counting:

- If the environment contains a tokenizer that matches the target model, provide a local estimate and state the tokenizer and source of error.
- If no matching tokenizer is available, do not present character count as an exact token count. Leave token statistics to the server-side dry run.

## 4. Query and confirm hyperparameters

| Command | When to use | Common parameters |
|---|---|---|
| `arkcli models finetune-config` | Query supported hyperparameters and `dataset_schema` after selecting the model version and customization type | `<model> <version>`, `--type` |
| `arkcli train finetune create` | Preview or create a job | Use `--dry-run` for preview, then submit after confirmation |

Display parameter names, default values, ranges or enum values, and short descriptions. Do not rely on memory for field names.

When `--type` is explicit, `models finetune-config` validates it against the exact model version's authoritative `FinetuneTypes`, using the same capability source as `train finetune capability get`. If that version does not support the requested type, the command returns a validation error before reading its configuration schema; do not continue with estimate or creation.

- When the user requests custom hyperparameters, ask the user to confirm the override values.
- Use the selected model, user preference, dataset size, target metrics, and logs to recommend hyperparameters. Use defaults when there is no evidence for a better configuration.
- Reject values outside the schema.

Read default values and ranges for general training configuration from the `finetune-config` schema. This reference records only stable semantics:

- Field names for epochs, learning rate, batch size, LoRA rank/alpha/dropout, DPO beta, RL steps, and similar settings can vary by training method.
- `save_model_limit` controls how many training artifacts are retained. Follow the current CLI/API for its default and maximum.
- Data fault tolerance limits and the shuffle seed are data configuration, not model hyperparameters. Display them separately in the preview.

## 5. Preview creation and validate configuration, data, tokens, and cost

| Command | When to use | Common parameters |
|---|---|---|
| `arkcli train finetune create` | Preview or create a job | Use `--dry-run` for preview, then submit after confirmation |

Assemble the command according to `arkcli train finetune create --help`, then run it with `--dry-run`. If local-file upload requires confirmation, obtain user authorization first and add `--yes` only as directed by the CLI.

The preview must summarize at least:

- Job name, model, version, fine-tuning type, and training method.
- If defaults are applied: "No fine-tuning type or training method was specified; defaulting to SFT + LoRA."
- For full fine-tuning: "ArkCLI currently cannot deploy artifacts produced by full fine-tuning; complete deployment in the console after training."
- Training and validation data sources.
- Custom and recommended hyperparameters, plus which remaining parameters use defaults.
- Server-reported sample or token information.
- Billing unit, unit price, and estimated cost.
- Non-default data fault tolerance limits, random seed, artifact-retention limit, and similar configuration.

If the dry run omits a field, state that it was not provided. Do not invent a value.

## 6. Obtain final confirmation and create

- After authentication, generate and reuse
  `ARKCLI_SKILL_FLOW_ID=ftf_<ULID>` for this create workflow.
- With the shared single-command prefix, run
  `arkcli train finetune _report-activity --action create_flow_enter` before
  the first create business command.
- Report `data_validation_success` only after machine-verifiable data validation;
  ordinary argument parsing and Client Preview `--dry-run` do not report.

Present the complete preview and explicitly ask whether to create the job. Run the real creation command only after confirmation. For non-interactive execution, add `--yes` as required by the CLI.

On success, return:

- Job ID, name, and initial phase.
- Model, version, fine-tuning type, and training method.
- A summary of key data and hyperparameters.
- Console URL, if returned by the CLI.
- A follow-up query such as `arkcli train finetune get <job-id>` or `watch <job-id>`.

Do not deploy the model automatically in this flow.
