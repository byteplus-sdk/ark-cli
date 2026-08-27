---
name: arkcli-datasets
description: Use ArkCLI to create, validate, inspect, update, delete, version, and download datasets. Trigger for Dataset, ds-*, dsv-*, reusable training or validation data, schema validation, and dataset-backed fine-tuning jobs.
---

# ArkCLI Datasets

Read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) first and follow its authentication, output, safety, Client Preview, and confirmation rules.

## Scope

- Create a Dataset and its first `V1`, or create a new version for an existing Dataset.
- Use local `.jsonl` files or map objects from one TOS bucket.
- Resolve the exact `DatasetSchema` from a foundation model fine-tuning configuration and run server-side validation.
- Inspect, update, delete, version, and download Dataset resources.
- Produce stable `<dataset-id>:<version-id>` references for `train finetune create`.

Use [`../arkcli-train-finetune/SKILL.md`](../arkcli-train-finetune/SKILL.md) when the task only manages fine-tuning jobs. Do not replace an available `dataset` product command with Raw API calls.

## Guards

1. Run `arkcli auth status` first and recover authentication through the shared skill if needed.
2. Read the target leaf command's `--help` before execution and follow the installed CLI version.
3. `--dataset-format` accepts only `LLMSft`, `LLMPretrain`, `LLMDPO`, `LLMRL`, `EmbeddingSft`, `ImageGenSft`, or `VideoGenSft`. Never send display labels or another backend SchemaType.
4. `--local` and `--tos-uri` are mutually exclusive. Local inputs must be `.jsonl` and are limited to 20 files. Multiple TOS URIs must use one bucket. Every `--tos-uri` prefix must be listable by the current identity and contain at least one `.jsonl` object. Never reuse a `ds-ds/*` upload staging path; use registered storage under `ark/dataset/<dataset-id>/<dataset-version-id>/` or another readable data path.
5. Create and version-create validate every input first. Omit `AccountID` from `ValidateDataset.StorageRawLocation.Storage`, then include the current account's numeric `AccountID` in `CreateDatasetVersion.Storage` for both V1 and later versions. `CreateDatasetVersion` omits the optional `Name`, and the CLI exposes no flag for it. Treat the response `Version` as authoritative. A failure does not roll back uploaded objects or an already-created Dataset.
6. `dataset version get <dataset-id> <Vn|dsv-*> --preview` reads at most 100 KiB (102400 bytes) from each object, matching the console Dataset preview limit. Downloads support both `dataset download ... --version ...` and `dataset version download <dataset-id> <Vn|dsv-*>`; neither overwrites existing files.
7. Before deletion, display version count, storage-prefix count, and object count/bytes when available, then confirm separately.
8. Create, version-create, validate, download, update, and delete support leaf-local `--dry-run`. Read-only list/get commands do not; `version get --preview` is a real read-only query, not Client Preview.
9. Client Preview is an offline plan. It does not upload data, validate server state, or replace delete confirmation.
10. BytePlus Dataset storage is supported only in AP regions. Stop locally when the active profile uses a non-AP region.

## Routing

- Read [`references/commands.md`](references/commands.md) for create, maintenance, validation, download, and fine-tuning references.
- After preparing a Dataset, read [`../arkcli-train-finetune/references/create.md`](../arkcli-train-finetune/references/create.md) for job creation.
- Maintainers can use [`references/evals.md`](references/evals.md) for regression coverage.

Load only the reference required by the current task.
