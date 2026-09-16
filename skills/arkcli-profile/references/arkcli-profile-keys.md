# `arkcli profile keys`

Read [`../SKILL.md`](../SKILL.md) first.

The `keys` subtree manages the API key inventory stored in one BytePlus
profile:

| Command | Remote call | Local change |
|---|---:|---:|
| `keys list` | Yes, reconciles against the profile's live key source | May write the reconciled inventory and select or clear the default when membership changed |
| `keys use <index|api-key>` | No | Changes the profile's default API key |
| `keys refresh` | Yes | Synchronizes the profile's available API keys |

`arkcli auth apikey` is a separate interactive identity-level key-selection
flow for the ordinary API-key pool. Do not substitute it for explicit Profile
inventory management or a Coding Plan Team seat key; `saved=true` does not
prove that a team profile is usable.

## List keys

Before executing `keys list`, explain that the command contacts the applicable
remote key source, reconciles the live result with the local inventory, and may
write the inventory or select/clear the local default. Masking protects output
secrets; it does not make this operation side-effect free.

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
| `coding-plan-team` | `GetSeatInfo.Result.ApiKey` from the assigned Running Coding Plan Team seat |

It does not rotate, create, or revoke a backend key.
If the applicable source returns zero usable keys, refresh fails and preserves
the existing local available-key inventory and default key. It must not report
success or clear local keys from an empty response.

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
  refreshing the team profile. Do not use `auth apikey`; the ordinary pool is
  unrelated to the seat key.
- Ordinary key inventory empty: a real TTY login can ask whether to create one
  all-resource key. Only explicit confirmation permits one creation followed
  by at most three read-only status checks; non-interactive execution and
  refusal perform no mutation. Manual recovery uses the BytePlus ARK ordinary
  API-key management page.
- Applicable remote source returns no usable key: refresh reports a
  type-specific error and preserves the local keys.
- Key lacks resource permission: select or create a backend key with the
  required permission. Refresh alone cannot grant access.
- Key removed remotely: refresh, inspect the new numbered list, and explicitly
  select another available key.

Never display a complete API key or read the local credential store directly.
