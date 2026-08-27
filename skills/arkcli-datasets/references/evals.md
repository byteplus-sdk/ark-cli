# arkcli-datasets evals

## Trigger

- Create, validate, inspect, update, delete, version, or download a Dataset.
- A request contains `ds-*`, `dsv-*`, or a friendly `Vn`.
- Reusable Dataset input is needed for a fine-tuning job.

## Anti-trigger

- Use `arkcli-train-finetune` for an existing `mcj-*` job with no Dataset management.
- Do not trigger for generic TOS uploads unrelated to Dataset resources.

## Regression cases

| case | prompt | expected behavior |
|---|---|---|
| `dataset-create-local` | Create an LLMSft Dataset from train.jsonl. | Preview with leaf-local `--dry-run`, then execute; omit `AccountID` from validation, include the current account in V1 `CreateDatasetVersion.Storage.AccountID` as a JSON number, and do not invent rollback. |
| `dataset-format-invalid` | Create a Dataset with `ImageGenerationSFT`. | Reject locally and list every valid value: `LLMSft`, `LLMPretrain`, `LLMDPO`, `LLMRL`, `EmbeddingSft`, `ImageGenSft`, and `VideoGenSft`. |
| `dataset-version-friendly-id` | Get V3 from ds-1. | Run `dataset version get ds-1 V3`; never send V3 without the Dataset ID. |
| `dataset-version-sid` | Get dsv-1. | Run `dataset version get dsv-1` without requiring a redundant Dataset ID. |
| `dataset-version-preview` | Preview V1 from ds-1. | Run `dataset version get ds-1 V1 --preview`; read no more than 100 KiB (102400 bytes) per object and return `truncated`. |
| `dataset-version-concurrent-create` | Concurrently create two versions for one Dataset. | Run them independently. `CreateDatasetVersion` omits the optional `Name` and does not derive one from `CreatedVersions`. Use the server-returned `Version` for each call. |
| `dataset-validate-schema` | Validate TOS data for a model/version DPO configuration. | Run model-aware `dataset validate`; do not use the type string as SchemaType or add `--sample-count`. |
| `dataset-tos-staging-prefix` | Create from `tos://.../ds-ds/<uuid>/` returned by an earlier local upload. | Reject before `ValidateDataset`, explain that `ds-ds/*` is upload staging, and direct the user to `ark/dataset/<dataset-id>/<dataset-version-id>/` or another listable prefix containing JSONL objects. |
| `dataset-download-conflict` | Download V2 into a directory containing the same target. | Preserve relative paths and stop without overwriting. |
| `dataset-version-download-alias` | Download ds-1/V2 from the version command. | Use `dataset version download ds-1 V2`; preserve the same Preview, output, and no-overwrite contract as `dataset download --version V2`. |
| `dataset-delete-impact` | Delete ds-1. | Display version count, storage-prefix count, and available object count/bytes before a separate confirmation. |
| `dataset-byteplus-region` | Create a Dataset from a non-AP BytePlus profile. | Stop locally with the AP-region constraint before any upload or Dataset API call. |
| `finetune-dataset-reference` | Train with ds-1/dsv-1 and one preset. | Use `--train-dataset ds-1:dsv-1` and wire-shape preset JSON; reject a simultaneous training TOS or local-file source. |
| `finetune-train-path-sampling` | Scale one Dataset by 2 and sample 500 rows from another. | Repeat `--train-path`; each entry uses only `multiplier` or `sample_count`, and no other training source may be combined. |
