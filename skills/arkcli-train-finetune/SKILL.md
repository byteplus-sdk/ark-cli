---
name: arkcli-train-finetune
description: Create, query, and manage model fine-tuning jobs in BytePlus ModelArk with ArkCLI. Use this skill for every request that names a fine-tuning job ID (`mcj-*`), including not-found diagnosis, logs, rollout trajectory, status, or lifecycle operations. Also use it for fine-tuning capability, pricing, configuration, job creation, metrics, artifact export, and deployment handoff. This skill does not manage datasets.
---

# ArkCLI Fine-Tuning for BytePlus ModelArk

First read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) and follow its authentication, output, safety, and confirmation rules.
Before running control-plane commands, confirm that the active profile, project, and region all belong to the intended BytePlus environment.

## When To Trigger and Scope

- Create a fine-tuning job: read [`references/create.md`](references/create.md).
- List or filter fine-tuning jobs: read [`references/list.md`](references/list.md).
- Retrieve, observe, or operate a specific job: read [`references/manage.md`](references/manage.md).
- Select a step by metrics, export artifacts, and deploy: read [`references/export-deploy.md`](references/export-deploy.md).
- Do not directly manage datasets. Accept local files, BytePlus TOS URIs, `ds-*/dsv-*` references, and model-supported presets. Use [`../arkcli-datasets/SKILL.md`](../arkcli-datasets/SKILL.md) to create or maintain Dataset resources.
- Use `--train-dataset` for an ordinary managed Dataset with the default `Multiplier=1`; use repeatable `--train-path` for multiple references, scaling, or sampling. Each entry sets at most one of `multiplier` or `sample_count`; omitting both keeps `Multiplier=1`. Each preset must choose exactly one of `inject_multiplier` or `inject_sample_count`.
- Orchestrate metric analysis and artifact export in this skill. For custom model details, preparation of deployable versions, and endpoint creation, follow the model registry and deployment skills.
- Do not use the low-level API Explorer as the default entry point.

Load only the reference required for the current task. Do not read every reference merely to become familiar with all commands.

## When NOT To Trigger

- Dataset-only management with no fine-tuning job work → use the dataset capability instead.
- Public foundation-model catalog lookup only → use [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md).
- Management of an existing inference Endpoint only → use [`../arkcli-infer-endpoint/SKILL.md`](../arkcli-infer-endpoint/SKILL.md).
- Authentication-only or profile/config troubleshooting → use [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md) or [`../arkcli-config/SKILL.md`](../arkcli-config/SKILL.md).
- The low-level API Explorer is a fallback entry only, never the default. Use it only when the product command cannot express the task and the user authorizes the fallback.

## Exact-scope diagnostics for a named job

- When a user provides an `mcj-*` ID and asks about status, why it cannot be found, logs, or trajectory, load this skill and read [`references/manage.md`](references/manage.md).
- For “why can’t this job be found?”, first run `arkcli train finetune get <mcj-id>` against the current active profile, project, and region, then explain the authoritative API response.
- Treat `mcj-*` as an opaque resource ID. Never reject it from a date-like segment, a readable suffix, or an invented hash-format rule, and never rewrite the supplied ID.
- If the user says not to switch environments or to inspect only this job, do not run `train finetune list`, scan other jobs, switch profile/project/region, or query another account. Preserve the target `get` error code, message, and request ID. Expand scope only after separate user authorization.
- When the user asks to save one named MCJ’s logs to a local path, the first business command is `arkcli train finetune logs <mcj-id> --output <path>`. Never list all jobs or scan them with a script first. Preserve any target-command error and do not suggest switching environments.
- When the user requests the complete rollout trajectory for one named MCJ, run `arkcli train finetune trajectory list <mcj-id> --full` directly. There is no `arkcli train trajectory` path. Preserve a no-trajectory or logging-disabled error; do not explore profiles, MCP configuration, or other jobs.
- `logs --follow` polls only while a job is active and may produce more logs. It emits the current snapshot and exits when the job is already terminal, and also exits after an idle poll observes a terminal phase. Do not rely on an external timeout as the normal termination mechanism.
- `pause` and `resume` form an explicit reversible pair: `pause` moves a running job to `Paused`, and `resume` restores a `Paused` job. When the backend permits, `resume` may also retry `Failed` or `Terminated`; follow the current API result.

## Live-information policy

Do not hard-code information that can change, including:

- Trainable models, model versions, fine-tuning types, and training methods.
- Fine-tuning prices.
- Hyperparameter fields, defaults, ranges, and enum values.
- CLI flags, job phases, and operation restrictions.
- Post-fine-tuning inference methods supported by base models or custom models.

Before critical commands, or when a command fails, use the installed version's `--help` and ArkCLI query commands to obtain current information. If CLI output differs from the command skeletons in this skill, follow the current CLI.

Query prices and fine-tuning capabilities under the active BytePlus profile. Even when model or training method names match those on another platform, do not reuse that platform's prices or capability results.

Treat BytePlus model selection as fail-closed: before naming or using a model, require its exact name from a live catalog query under the active BytePlus scope, then verify its version and requested fine-tuning type with `models versions` and `train finetune capability get`. Treat names from the user, documentation, `--help`, prior knowledge, or another product or region as unverified; never normalize product prefixes. If either query fails or returns no exact eligible match, stop model-dependent estimation or creation instead of guessing a fallback. Continue using `--help` for syntax.

Use the BytePlus ModelArk [Model fine-tuning dataset format description](https://docs.byteplus.com/en/docs/ModelArk/1099461) as the primary data-format source, then confirm with model-aware server-side validation. Do not maintain format descriptions or examples here that are likely to become outdated.

## Default fine-tuning type, training method, and deployment limitation

- If the user does not specify a fine-tuning type (`--type`), default to SFT.
- If the user does not specify LoRA or full fine-tuning as the training method, default to LoRA.
- If the user explicitly selects full fine-tuning, explain before creation that ArkCLI currently cannot deploy artifacts produced by full fine-tuning and that deployment must be completed in the console after training.

## BytePlus fine-tuning SDK availability

BytePlus currently does not support the fine-tuning SDK. If the user requests a custom grader, rollout plugin, SDK workspace, custom training code, or SDK submission, politely explain that the capability is currently unavailable on BytePlus. If standard ArkCLI parameters or pipeline configuration can fully express the request, continue with the standard ArkCLI creation flow. Rollout trajectory queries are a separate ArkCLI capability: for a job created with `--enable-trajectory`, use `arkcli train finetune trajectory` with an SSO-authenticated profile that can access the corresponding TLS topic.

## Critical client-side validation

- When the user supplies a model name and fine-tuning type and asks for fine-tuning, SFT, or LoRA pricing, first run `arkcli train finetune pricing --model <model> --type <type>`. Never substitute generic `arkcli pricing models`, because the generic billing catalog does not cross-check the requested training capability for that model.
- When the user explicitly selects a fine-tuning type, call `models finetune-config <model> <version> --type <type>` with the exact model version. The command validates the type against that version's authoritative `FinetuneTypes`; stop on an unsupported type instead of continuing to pricing, estimate, or job creation.
- For manual pagination, `train finetune list --page-number` must be `>=1` and `--page-size` must be within `1-100`. For page 2 or later, a page beyond the filtered `total_count` is a parameter error; do not treat it as a valid empty page.
- When both `train finetune metrics --from-step` and `--to-step` are present, `to-step` must be strictly greater than `from-step`. Stop on an invalid interval before querying metric names or curves.
- `train finetune pricing --billing-method token` queries token charge items. `instance` requires the exact `--model-version` and `--type`, with hyperparameters kept consistent with job creation. Treat the hourly range as complete only when `price_complete=true`; otherwise report `missing_flavor_ids`.

## Guard Checklist and General Execution Rules

1. Run `arkcli auth status` first. If authentication fails, recover through the shared skill.
2. Run read operations directly. For file uploads, job creation, billable operations, or destructive operations, follow the confirmation rules.
3. Do not ask again for parameters the user already specified. Ask only when a required value is missing and cannot be derived from live queries.
4. Distinguish the source of every fact: CLI/API responses, server-side validation, or a local rough estimate.
5. Do not print credentials, complete training logs, or other large results. Write large results to a file and extract only the required fields.

## References and Related Documentation

- [Model fine-tuning overview](https://docs.byteplus.com/en/docs/ModelArk/1099459)
- [Model fine-tuning dataset format description](https://docs.byteplus.com/en/docs/ModelArk/1099461)

Maintainer evaluation cases are in [`references/evals.md`](references/evals.md).
