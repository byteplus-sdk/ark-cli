# `arkcli profile keys`

Read [`../SKILL.md`](../SKILL.md) first.

The `keys` subtree manages the API key inventory stored in one BytePlus
profile:

| Command | Remote call | Local change |
|---|---:|---:|
| `keys list` | No | No |
| `keys use <index|api-key>` | No | Changes the profile's default API key |
| `keys refresh` | Yes | Synchronizes the profile's available API keys |

`arkcli auth apikey` is a separate interactive identity-level key-selection
flow. Do not use it for read-only inventory.

## List keys

```bash
arkcli profile keys list --format json
arkcli profile keys list \
  --profile platform_ap-southeast-1_accountwide \
  --format json
```

Keys are always masked in output. The response includes a numbered view for
safe selection:

```json
{
  "profile": "platform_ap-southeast-1_accountwide",
  "default_api_key": "392****dab0",
  "available_api_keys": [
    "392****dab0",
    "abc****1234"
  ],
  "keys": [
    {
      "index": 1,
      "api_key": "392****dab0",
      "is_default": true
    },
    {
      "index": 2,
      "api_key": "abc****1234",
      "is_default": false
    }
  ]
}
```

## Select the default key

Prefer the 1-based index returned by `keys list`:

```bash
arkcli profile keys use 2
arkcli profile keys use 2 --profile <profile-name>
```

A complete, unmasked key is also accepted for compatibility:

```bash
arkcli profile keys use <complete-api-key>
```

The selected key must already exist in the profile's available-key inventory.
Never pass a masked value as a key. On success, output contains the profile name
and masked `new_default`.

## Refresh keys

```bash
arkcli profile keys refresh
arkcli profile keys refresh --profile <profile-name>
```

Refresh uses the target profile's BytePlus identity and the key source required
by its type:

| Profile type | Key source |
|---|---|
| `platform` | Platform API key inventory |
| `coding-plan` | Platform API key inventory used by personal Coding Plan |
| `coding-plan-team` | API key associated with the assigned Coding Plan Team seat |

It does not rotate, create, or revoke a backend key.

Example output:

```json
{
  "profile": "platform_ap-southeast-1_accountwide",
  "default_api_key": "392****dab0",
  "available_api_keys": [
    "392****dab0",
    "abc****1234"
  ],
  "added_count": 1,
  "removed_count": 0,
  "refresh_status": "ok"
}
```

## Recovery

- SSO or STS expired: recover BytePlus login, then retry refresh once.
- `ARK_API_KEY` or `--api-key` overrides the stored default: remove the runtime
  override if it is unintended; refreshing the profile does not override it.
- Coding Plan Team seat unavailable: restore or assign the seat before
  refreshing the team profile.
- Key lacks resource permission: select or create a backend key with the
  required permission. Refresh alone cannot grant access.
- Key removed remotely: refresh, inspect the new numbered list, and explicitly
  select another available key.

Never display a complete API key or read the local credential store directly.
