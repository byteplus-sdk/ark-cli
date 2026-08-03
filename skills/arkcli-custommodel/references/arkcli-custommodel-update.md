# arkcli models custommodel update

Update a custom model's name or description.

## Usage

```bash
arkcli models custommodel update <id> [flags]
```

## Examples

```bash
# Update the name
arkcli models custommodel update cm-xxxxx --name new-display-name

# Update the description
arkcli models custommodel update cm-xxxxx --description "updated for v2 release"

# Update both fields
arkcli models custommodel update cm-xxxxx --name v2 --description "..."
```

## Flags

| Parameter | Required | Type | Description |
|---|---|---|---|
| `<id>` | Yes | string | Custom model ID in `cm-xxxxx` form |
| `--name` | No* | string | New account-unique name. It must begin with an ASCII letter; subsequent characters may be Chinese characters, ASCII letters, digits, underscores, or hyphens. |
| `--description` | No* | string | New description, up to 300 characters |

*Pass at least one of `--name` or `--description`.

**Note:** This command changes **display properties only**. It cannot change the base model, customization type, quantization method, or status. Changing `--name` does not affect deployed endpoint references because endpoints reference the model by ID.

## Output

On success, `result` contains only `id` and the fields changed by this request:

```json
{
  "id": "cm-xxxxx",
  "name": "new-display-name"
}
```

The response contains both `name` and `description` only when both fields were changed. It does not return unchanged fields or the complete model details.
