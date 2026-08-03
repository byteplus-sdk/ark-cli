# arkcli models custommodel list

List custom models with pagination and multidimensional filters.

## Usage

```bash
arkcli models custommodel list [flags]
```

## Examples

```bash
# List all custom models with default pagination
arkcli models custommodel list

# List only custom models created by the current SSO sub-user
arkcli models custommodel list --mine

# Filter by status; use comma-separated values or repeat the flag
arkcli models custommodel list --statuses ready
arkcli models custommodel list --statuses ready,processing

# Filter by customization type
arkcli models custommodel list --customization-types lora,sft

# Filter by base model name or ID
arkcli models custommodel list --base-models <byteplus-base-model-name>
arkcli models custommodel list --base-model-ids fm-xxxxx

# Fuzzy-search name, ID, or base model display name
arkcli models custommodel list --search my-finetune

# Paginate and sort
arkcli models custommodel list --page 1,20 --sort-by CreateTime --sort-order desc

# Paginate automatically
arkcli models custommodel list --mine --page-all --page-delay 500
```

## Flags

> Every multivalue filter of type `string list` accepts either comma-separated values (`--statuses ready,processing`) or repeated flags (`--statuses ready --statuses processing`). Both forms produce the same result.

| Parameter | Required | Type | Description |
|---|---|---|---|
| `--mine` | No | bool | Return only custom models created by the current SSO sub-user |
| `--statuses` | No | string list | `preparation` / `processing` / `ready` / `failed` / `exporting` / `exportfailed` |
| `--customization-types` | No | string list | `lora` / `sft` / `dpolora` / `dpo` / `grpolora` / `grpo` / `ppo` / `opdlora` / `opd` / `pretrain` |
| `--supported-customization-types` | No | string list | Filter by customization types supported by the underlying base model |
| `--base-models` | No | string list | Filter by base model name |
| `--base-model-ids` | No | string list | Filter by base model ID |
| `--sources` | No | string list | `import` / `customization` |
| `--source-jobs` | No | string list | Filter by fine-tuning job ID |
| `--search` | No | string | Fuzzy-search name, ID, or base model display name |
| `--page` | No | string | Pagination expression in `<number,size>` form, such as `1,10`; spaces are not accepted |
| `--sort-by` | No | string | Sort field; defaults to `CreateTime` |
| `--sort-order` | No | string | Sort direction; only `asc` / `desc` are accepted; defaults to `desc` |

## Output

- `--format json` and `--format yaml` preserve the complete execution and pagination response.
- `--format table` and `--format csv` expand the list to one row per `result.items[]` entry instead of rendering the whole `result` as a map.

Each item commonly includes `id` (`cm-xxxxx`), `name`, `status`, `foundation_model`, `customization_type`, `source_type`, `create_time`, and `update_time`. Nested table cells such as `foundation_model` use compact JSON.
