# models get

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

View model details, aggregating multiple underlying model metadata APIs to return complete information.

## Commands

```bash
# Positional argument form (recommended)
arkcli models get dola-seed-2-1-turbo-260628

# Specify a version
arkcli models get dola-seed-2-1-turbo-260628 260628

# Explicit flag form
arkcli models get --id dola-seed-2-1-turbo-260628 --version 260628
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `<id>` / `--id` | Yes | string | Model identifier, such as `dola-seed-2-1-turbo-260628` |
| `[version]` / `--version` | No | string | Model version override, such as `260628` |

## Return value

Model details in JSON format, aggregated from multiple underlying APIs. Includes model name, version, capabilities, pricing, rate limits, and other information.

## Common errors

| Error | Cause | Handling |
|------|------|---------|
| Model does not exist | The ID is misspelled or the model has been taken offline | Use `arkcli models search` to confirm the model name |
| Authentication failed | Not logged in or credentials expired | Run `arkcli auth login` to re-establish BytePlus identity |

## Notes

- Both `id` and `version` support positional arguments and flags.
- This command aggregates multiple underlying APIs and may be slightly slower than `list`.

## Guards

- If authentication fails, first return to `arkcli auth status`, then run `arkcli auth login`.
- If the model ID is uncertain, first use `arkcli models search` or `arkcli models list --name` to confirm it and avoid repeatedly calling a nonexistent ID.
- Prefer read-only queries. Do not switch profiles or modify local configuration while troubleshooting model details unless the user explicitly confirms.

## References

- [arkcli-models](../SKILL.md) -- All models commands
- [arkcli-shared](../../arkcli-shared/SKILL.md) -- Authentication and global parameters
