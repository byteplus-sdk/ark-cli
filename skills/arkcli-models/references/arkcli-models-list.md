# models list

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

List the BytePlus ModelArk public foundation-model catalog with pagination, modality filtering, and sorting. It is suitable for public catalog enumeration and statistics. If the user is looking for "which model is suitable for a task", still prefer `arkcli models search`.

## Commands

```bash
# List all models (paginated by default)
arkcli models list

# Filter by modality
arkcli models list --modality text

# Case-insensitive substring filtering by name
arkcli models list --name dola

# Pagination control
arkcli models list --page-size 20 --page-number 2

# Sorting (sort-order must be Asc/Desc with an uppercase first letter)
arkcli models list --modality text --sort-by UpdateTime --sort-order Desc

# Retrieve the complete list for local script statistics/filtering
arkcli models list --page-all --sort-by CreateTime --sort-order Desc --format json
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `--modality` | No | string | Filter by modality: `text` / `image` / `video` / `audio` / `embed` |
| `--name` | No | string | Case-insensitive substring filtering by model name |
| `--page-number` | No | int | Page number (>=1) |
| `--page-size` | No | int | Number of entries per page |
| `--sort-by` | No | string | Sort field, such as `UpdateTime`, `CreateTime` |
| `--sort-order` | No | string | Sort direction. Valid values: `Asc` / `Desc` (uppercase first letter) |

## Public catalog versus account assets

`models list` enumerates public foundation models, not account-owned custom models or deployed Endpoints.

- For "my custom models" or recent custom-model creation counts, use [arkcli-custommodel](../../arkcli-custommodel/SKILL.md) and `arkcli models custommodel list --mine --page-all --format json`. Remove the personal scope only when explicitly asked for account-wide assets. Do not add `--ready` when counting all created models.
- Filter actual asset creation timestamps in the user's timezone and time window. Missing timestamps or incomplete pagination mean unknown/partial counts. Never infer ownership from public catalog names, tags, model types, or creation timestamps.
- Deployed resources belong to `infer endpoint list`; currently callable resources belong to `resources list`. Coding Plan catalogs use `plans model-list` for the explicit plan. These sets are not interchangeable.
- Missing capability fields in `models list` are not negative evidence. Use exact-model/version `models search/get` metadata. An empty public catalog does not prove that the account has no custom assets or Endpoints.

`--transform` extracts fields such as `items.#` or `items.#.name`; it is not full jq and cannot perform date arithmetic. Retain complete JSON before local projection/filtering, and do not count a truncated sample as the full inventory.

## Return value

A paginated JSON result with top-level fields `page_number`, `page_size`, `total_count`, and `items`. Each item normally includes fields such as `name`, `display_name`, `primary_version`, `access_type`, and `foundation_model_tag`.

## Common errors

| Error | Cause | Handling |
|------|------|---------|
| Empty result | The `--name` substring matched no public models | Use `arkcli models search` for fuzzy search |
| Authentication failed | Not logged in or credentials expired | Run `arkcli auth login` to re-establish BytePlus identity |

## Notes

- `--name` is a case-insensitive substring filter. For exact details, use `models get <id>`.
- Combine with `--transform` to extract a field, such as `--transform 'items.0.name'`.
- Select the correct public-catalog, custom-asset, Endpoint, or Plan command before pagination and filtering. A missing filter does not justify querying another resource class or guessing a Raw API.

## References

- [arkcli-models](../SKILL.md) -- All models commands
- [arkcli-shared](../../arkcli-shared/SKILL.md) -- Authentication and global parameters
