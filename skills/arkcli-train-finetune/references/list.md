# List Fine-Tuning Jobs

Use this reference only for listing and filtering multiple jobs. If the user provides a job ID, read [`manage.md`](manage.md) instead.

## Command and common filters

```bash
arkcli train finetune list [flags]
```

| Parameter | Description |
|---|---|
| `--name` | Fuzzy-filter by job name |
| `--phase` | Filter by job phase; repeatable |
| `--customization-type` | Filter by backend customization type; repeatable. Check `--help` when enum values are uncertain. |
| `--create-time-after`, `--create-time-before` | Filter by creation time using RFC3339 |
| `--mine` | Only include jobs created by the current SSO sub-user; server-side `TagFilters=[{Key:"sys:ark:createdBy", Values:["IAMUser/<uid>/<name>"]}]` |
| `--page-size`, `--page-number` | Paginate manually. Page number defaults to 1 and must be `>=1`; page size defaults to 100 and must be in `1-100`. |
| `--page-all`, `--page-limit` | Paginate automatically; always set a reasonable limit |
| `--sort-by`, `--sort-order` | Sort results |
| `--transform` | Extract a single field; use a JSON tool for complex projections |

## Workflow

1. Inspect parameters supported by the current version:

```bash
arkcli train finetune list --help
```

2. Combine filters according to the user's criteria:

- Name.
- Job phase.
- Customization type.
- Creation time range.
- Creator. When the user says "mine / my jobs / jobs I created", add `--mine`; do not list the whole account and guess the creator locally.
- Pagination and sorting.

3. If the user requests all results, use the current CLI's automatic pagination parameters and set a reasonable page limit. Never fetch without a bound.

For manual requests starting at page 2, the CLI first obtains `total_count` using the active filters. It returns a validation error when the requested page starts beyond the total; a final partially filled page remains valid.

4. Return a concise table or summary that prioritizes:

- Job ID.
- Name.
- Customization type or training method, as returned.
- Current phase.
- Creation or update time.

Do not automatically call `get` for every listed job. Query a job only when the user requests details or the list response lacks a field needed to answer the request.

## Command skeletons

```bash
arkcli train finetune list
arkcli train finetune list --name <keyword>
arkcli train finetune list --phase <phase>
arkcli train finetune list --mine --page-all --page-size 100
arkcli train finetune list \
  --create-time-after <RFC3339> \
  --create-time-before <RFC3339>
```

Obtain phase and customization-type enum values from the current `--help`; do not maintain a static list.

`--mine` depends on an SSO sub-user identity and the `sys:ark:createdBy` resource tag. A degraded identity
without an ARK IAM `UserID` gets an explicit unsupported error. Do not fake an empty tag or silently return
account-wide rows as "mine".

## Time handling

- When the user provides relative time such as "today," "yesterday," or "the last week," convert it to explicit RFC3339 boundaries using the active profile or user timezone.
- Include the actual absolute time range in the result to avoid timezone ambiguity.

## Empty results

For an empty result, state the filters actually used and suggest relaxing the filter most likely to be too restrictive. Do not interpret an empty list as a permission problem unless the CLI explicitly returns an authentication or authorization error.
