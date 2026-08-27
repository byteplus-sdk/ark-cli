# Dataset command workflows

## Create and version

Preview first, then execute after the required confirmation:

```bash
arkcli dataset create --name <name> --dataset-format LLMSft --local <train.jsonl> --dry-run
arkcli dataset create --name <name> --dataset-format LLMSft --local <train.jsonl>
arkcli dataset create --name <name> --dataset-format LLMSft --tos-uri tos://<bucket>/<path>
```

Each `--tos-uri` prefix must be listable by the current identity and contain at least one `.jsonl` object. Do not reuse `ds-ds/<uuid>/...` from local-upload output: it is upload staging and is not a mapped source for the Dataset validation engine. Prefer registered storage under `ark/dataset/<dataset-id>/<dataset-version-id>/` or another known-readable data path. The CLI lists the prefix before submitting validation and distinguishes an unreadable listing from a prefix with no JSONL objects.

Save `dataset_id`, `dataset_version_id`, `version`, `storage`, and `validation_jobs` from structured output. Local objects use `ds-ds/<uuid>/<filename>`. `ValidateDataset.StorageRawLocation.Storage` omits `AccountID`; V1 and later `CreateDatasetVersion.Storage` requests include the current account's numeric `AccountID`.

```bash
arkcli dataset version create <dataset-id> --local <new.jsonl> --dry-run
arkcli dataset version create <dataset-id> --local <new.jsonl>
```

Real execution still reads the Dataset for the SchemaType needed by validation, but it does not derive a name from `CreatedVersions` or send the optional `CreateDatasetVersion.Name`. The CLI exposes no flag for that field, and Client Preview omits `Name` as well. Treat the response `Version` as authoritative; the server handles concurrent version allocation.

## Inspect and maintain

```bash
arkcli dataset list --page-number 1 --page-size 100
arkcli dataset get <dataset-id>
arkcli dataset update <dataset-id> --name <new-name> --dry-run
arkcli dataset update <dataset-id> --name <new-name>
arkcli dataset delete <dataset-id> --dry-run
arkcli dataset delete <dataset-id>
arkcli dataset version list <dataset-id>
arkcli dataset version get <dataset-version-sid>
arkcli dataset version get <dataset-id> V2
arkcli dataset version get <dataset-id> V2 --preview
```

A friendly `Vn` needs its Dataset ID. A `dsv-*` SID can be queried directly. `--preview` reads at most 100 KiB (102400 bytes) per object, matching the console, and reports truncation. Delete first displays the version, prefix, and available object impact before asking for confirmation; never bypass this with Raw API.

## Validate

```bash
arkcli dataset validate --local <data.jsonl> --model <model> --model-version <version> --type sft --dry-run
arkcli dataset validate --local <data.jsonl> --model <model> --model-version <version> --type sft
```

Real execution resolves the model's exact schema, prepares storage, submits one server-side validation per path, and polls each job to `Succeed` or `Failed`. Do not derive a schema from the type string and do not add a nonexistent `--sample-count` flag.

## Download

```bash
arkcli dataset download <dataset-id> --version <dsv-id-or-vn> --dry-run
arkcli dataset download <dataset-id> --version V2 --output-dir ./dataset-copy
arkcli dataset version download <dataset-id> <dsv-id-or-vn> --dry-run
arkcli dataset version download <dataset-id> V2 --output-dir ./dataset-copy
```

The default directory is `./<dataset-id>-<version>/`. Relative object paths are preserved and existing targets are never overwritten.

## Use with fine-tuning

```bash
arkcli train finetune create \
  --name <job-name> \
  --model <model> --model-version <version> --type sft \
  --train-dataset <dataset-id>:<dataset-version-id>
```

Use `--validation-dataset <dataset-id>:<dataset-version-id>` for validation data. For multiple ordinary Dataset references, scaling, or sampling, repeat `--train-path`. Each JSON object sets at most one of `multiplier` or `sample_count`; omitting both keeps `Multiplier=1`:

```bash
--train-path '{"dataset_id":"ds-...","dataset_version_id":"dsv-...","multiplier":1}'
--train-path '{"dataset_id":"ds-...","dataset_version_id":"dsv-...","sample_count":500}'
```

Presets also use repeatable JSON and must set exactly one of `inject_multiplier` or `inject_sample_count`:

```bash
--preset-dataset '{"dataset_version_id":"dsv-...","inject_sample_count":100}'
```

`--train-path` is mutually exclusive with every other training Dataset, TOS, or local-file source. The CLI verifies schema compatibility and rejects presets not declared by the model configuration.
