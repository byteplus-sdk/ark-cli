# Download Fine-Tuning Job Data

When the user wants to recover the training or validation data used by a fine-tuning job:

```bash
arkcli train finetune download <job-id> [--set all|train|validation] [--output-dir <dir>]
```

The command resolves `Data.TrainingSet` / `Data.ValidationSet` from
`GetModelCustomizationJob`, then downloads the referenced objects from TOS. The default output directory
is `./<job-id>-data`; by default both training and validation data are downloaded.

## Supported sources

| Job Data field | Behavior |
|---|---|
| `TosBucket` + `TosPaths` | Download the original TOS objects and preserve relative paths |
| `Datasets[]` (`DatasetId` + `DatasetVersionId`) | Resolve the Dataset version storage, then download its `Storage.Bucket` / `Storage.Prefixes` |
| `PresetDatasets[]` | Platform-managed preset data, not downloadable; reported under `skipped` |
| `LocalFiles[]` | Only creation-time local paths are recorded; file contents are not retained server-side and are reported under `skipped` |

When a set has multiple sources, the command writes them below separate `train/` or `validation/`
subdirectories and adds `01/`, `02/`, ... when needed to avoid filename collisions.

## Examples

```bash
# Download training and validation data.
arkcli train finetune download mcj-20260722153822-jqvd

# Download only the training set.
arkcli train finetune download mcj-20260722153822-jqvd --set train --output-dir ./mcj-data

# Download only the validation set.
arkcli train finetune download mcj-20260722153822-jqvd --set validation
```

The result includes each file's `object_key`, local `path`, and `size_bytes`, plus the bucket and prefixes
used for each set.

## Boundaries

- This is a control-plane job lookup plus TOS read, not a data-plane call. Do not pass `--api-key` or
  `--base-url`.
- The command fails if a target file already exists; choose a different `--output-dir` instead of silently
  overwriting local data.
- A job that only has `PresetDatasets` or `LocalFiles` has no downloadable TOS objects and returns a clear
  "no downloadable data" error.
- Do not guess a data location from the job name, model name, or an old directory. Use the job's
  `TosBucket` / `TosPaths` or the referenced Dataset version.
