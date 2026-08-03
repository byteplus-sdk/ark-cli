# `arkcli profile set-default`

Read [`../SKILL.md`](../SKILL.md) first.

This command writes one default resource ID into a selected BytePlus profile.
It does not create, deploy, activate, or purchase the resource.

## Syntax and examples

```bash
arkcli profile set-default <id> \
  [--profile <profile-name>] \
  [--modality <text|image|video>]
```

```bash
# Platform text Endpoint
arkcli profile set-default ep-xxxxxxxx \
  --profile platform_ap-southeast-1_accountwide \
  --modality text

# Personal Coding Plan text model
arkcli profile set-default ark-code-latest \
  --profile coding-plan_ap-southeast-1_personal \
  --modality text

# Coding Plan Team text model
arkcli profile set-default ark-code-latest \
  --profile coding-plan-team_ap-southeast-1_team \
  --modality text
```

The modality defaults to `text`.

## Inline availability verification

By default, Ark CLI lists resources with the target profile's identity and
accepts the ID only when it appears in that availability result:

| Profile type | `text` verification | `image` or `video` verification |
|---|---|---|
| `platform` | Platform Endpoint inventory | Platform Endpoint inventory |
| `coding-plan` | Personal Coding Plan model inventory | Platform Endpoint inventory in the same account |
| `coding-plan-team` | Coding Plan Team model inventory | Platform Endpoint inventory in the same account |

If verification rejects the ID, list valid values first:

```bash
arkcli resources list \
  --profile <profile-name> \
  --modality <text|image|video> \
  --format json
```

`--skip-verify` bypasses only the control-plane availability check. Use it only
after the user explicitly accepts that Ark CLI cannot verify the ID. It does
not create the resource, grant permission, or change product support.

## Output

```json
{
  "profile": "coding-plan_ap-southeast-1_personal",
  "modality": "text",
  "new_default": "ark-code-latest",
  "verified": true
}
```

`verified=false` means the command was run with `--skip-verify`; it does not
describe BytePlus account-opening or payment verification.

## Recovery

- Authentication or STS failure during inline verification: recover BytePlus
  SSO and retry once.
- ID not found: use `arkcli resources list` with the same profile and modality.
- No saved default: pass `--model` explicitly to the original inference command
  or select a valid resource after confirmation.
- Wrong profile: rerun with an explicit `--profile`; do not switch product,
  account, or Region as a fallback.
