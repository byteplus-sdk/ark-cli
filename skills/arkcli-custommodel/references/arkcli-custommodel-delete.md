# arkcli models custommodel delete

Delete a custom model. This operation is destructive and irreversible.

## Usage

```bash
arkcli models custommodel delete <id> [--yes] [--dry-run]
```

## Examples

```bash
# Preview the deletion request without deleting
arkcli models custommodel delete cm-xxxxx --dry-run

# Delete interactively with a [Y/N] confirmation prompt
arkcli models custommodel delete cm-xxxxx

# Delete non-interactively and skip the local confirmation prompt
arkcli models custommodel delete cm-xxxxx --yes
```

## Flags

| Parameter | Required | Type | Description |
|---|---|---|---|
| `<id>` | Yes | string | Custom model ID in `cm-xxxxx` form |
| `--yes` | No | bool | Skip the interactive [Y/N] confirmation |
| `--dry-run` | No | bool | Command-local Client Preview; makes no request, deletes nothing, and skips confirmation |

**Notes:**

- Deletion is **irreversible**. The same ID will not be reused, and references from deployed endpoints will become invalid immediately.
- Before deletion, run `custommodel get <id> --transform id,name,active_endpoints`, list all references, and restate the impact to the user.
- Add `--yes` only after the user has explicitly confirmed deletion of the target `cm-xxxxx` and understands that a non-empty `active_endpoints` value means the operation will break an inference path.
- If the user has not explicitly asked for deletion and only asks whether deletion is possible or requests a risk check, never execute the deletion.
