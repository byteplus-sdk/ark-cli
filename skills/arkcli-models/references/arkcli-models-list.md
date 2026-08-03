# models list

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

List the BytePlus ModelArk model catalog with pagination, modality filtering, and sorting. It is suitable for full enumeration, statistics, and asset inventory. If the user is looking for "which model is suitable for a task", still prefer `arkcli models search`.

## Commands

```bash
# List all models (paginated by default)
arkcli models list

# Filter by modality
arkcli models list --modality text

# Exact filtering by name
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
| `--name` | No | string | Exact filtering by model name |
| `--page-number` | No | int | Page number (>=1) |
| `--page-size` | No | int | Number of entries per page |
| `--sort-by` | No | string | Sort field, such as `UpdateTime`, `CreateTime` |
| `--sort-order` | No | string | Sort direction. Valid values: `Asc` / `Desc` (uppercase first letter) |

## Custom models / recently created statistics

When the user asks "my custom models", "how many custom models were created in the past seven days", or "list them", this is an asset inventory request. Use `models list` and do not switch to `arkcli api --list` to explore Raw APIs.

Recommended process:

1. First confirm authentication status: `arkcli auth status`.
2. Retrieve the complete list: `arkcli models list --page-all --sort-by CreateTime --sort-order Desc --format json`.
3. If the current version provides a custom-model type flag, prefer that flag. Otherwise, filter the local JSON using available fields such as `model_type`, `type`, `customization_type`, `source_type`, `customized_tags`, and `create_time`.
4. If there is no server-side time-window flag, compare dates locally using `create_time`.

Example:

```bash
arkcli models list --page-all --sort-by CreateTime --sort-order Desc --format json > /tmp/ark-models.json
python3 - <<'PY'
import json
from datetime import datetime, timedelta, timezone

data = json.load(open("/tmp/ark-models.json"))
items = data.get("items") or []
cutoff = datetime.now(timezone.utc) - timedelta(days=7)

def parse_time(value):
    if not value:
        return None
    value = value.replace("Z", "+00:00")
    try:
        return datetime.fromisoformat(value)
    except ValueError:
        return None

def is_custom(item):
    haystack = " ".join(str(item.get(k, "")) for k in (
        "model_type",
        "type",
        "customization_type",
        "source_type",
    ))
    tags = item.get("customized_tags") or []
    haystack += " " + " ".join(str(tag) for tag in tags)
    return "custom" in haystack.lower() or "customization" in haystack.lower()

matched = []
for item in items:
    created = parse_time(item.get("create_time") or item.get("CreateTime"))
    if created and created.tzinfo is None:
        created = created.replace(tzinfo=timezone.utc)
    if created and created >= cutoff and is_custom(item):
        matched.append(item)

print(json.dumps({
    "count": len(matched),
    "items": matched,
}, ensure_ascii=False, indent=2))
PY
```

`--transform` is suitable only for lightweight extraction and counting, such as `--transform 'items.#'` and `--transform 'items.#.name'`. The current transform is not full jq and does not support date operations or predicates such as `?(@.create_time)`. For time windows, use `--format json`, followed by `python3` / `jq` for client-side filtering.

## Return value

A paginated JSON result with top-level fields `page_number`, `page_size`, `total_count`, and `items`. Each item normally includes fields such as `name`, `display_name`, `primary_version`, `access_type`, and `foundation_model_tag`.

## Common errors

| Error | Cause | Handling |
|------|------|---------|
| Empty result | Exact `--name` matching found no result | Use `arkcli models search` for fuzzy search |
| Authentication failed | Not logged in or credentials expired | Run `arkcli auth login` to re-establish BytePlus identity |

## Notes

- `--name` is an exact match. For fuzzy search, use `arkcli models search`.
- Combine with `--transform` to extract a field, such as `--transform 'items.0.name'`.
- For statistics/inventory, do not fall back to Raw API Explorer just because a server-side filter is missing. Prefer `--page-all --format json`, followed by local filtering.

## References

- [arkcli-models](../SKILL.md) -- All models commands
- [arkcli-shared](../../arkcli-shared/SKILL.md) -- Authentication and global parameters
