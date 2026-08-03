# `arkcli profile create`

Read [`../SKILL.md`](../SKILL.md) first.

Profile creation writes local BytePlus configuration and may query the control
plane for projects, API keys, subscriptions, or a Coding Plan Team seat. It
does not replace BytePlus SSO.

## Supported types

| Type | Requirements |
|---|---|
| `platform` | A BytePlus identity; an API key is optional at creation time. |
| `coding-plan` | An active personal Coding Plan subscription, detected automatically unless `--plan-tier` is supplied. |
| `coding-plan-team` | An active Coding Plan Team seat. |

No other profile type is supported by the BytePlus build.

## Examples

```bash
# Create a Platform profile in the default project.
arkcli profile create --type platform --set-default

# Create a Platform profile for a named project.
arkcli profile create \
  --type platform \
  --project my-project \
  --set-default

# Create a personal Coding Plan profile and use detected subscription data.
arkcli profile create --type coding-plan --set-default

# Override personal Coding Plan tier detection only when explicitly required.
arkcli profile create \
  --type coding-plan \
  --plan-tier lite \
  --set-default

# Create a Coding Plan Team profile from the current assigned seat.
arkcli profile create --type coding-plan-team --set-default

# Fail instead of prompting when required input is missing.
arkcli profile create \
  --type platform \
  --project default \
  --no-interactive \
  --set-default
```

The Region is fixed to `ap-southeast-1`; it normally does not need to be passed.

## Flags

| Flag | Required | Meaning |
|---|---:|---|
| `--type` | In non-interactive mode | `platform`, `coding-plan`, or `coding-plan-team`. |
| `--region` | No | Fixed BytePlus Region: `ap-southeast-1`. |
| `--project` | No | Named project; manual creation falls back to `default`. |
| `--name` | No | Internal profile name; otherwise Ark CLI derives one. |
| `--default-api-key` | No | API key to bind as the profile default. |
| `--owner-trn` | No | Optional owner TRN; normally left empty for identity self-healing. |
| `--plan-tier` | No | Personal Coding Plan override: `lite` or `pro`. |
| `--set-default` | No | Make the new profile the local default. |
| `--no-interactive` | No | Disable prompts and fail on missing required input. |

Do not pass the internal account-wide sentinel to `--project`. To select
`All account resources`, use the interactive project selector:

```bash
arkcli profile project
```

That command replaces the existing Platform profile, so follow the confirmation
rules in the parent Skill.

## Result and recovery

- No API key: the profile may still be created, but data-plane commands remain
  unavailable until a key is selected.
- SSO or control-plane error: inspect
  `arkcli auth status --format json`, recover BytePlus SSO if required, and
  retry once.
- Personal Coding Plan not detected: verify the subscription. Use
  `--plan-tier lite|pro` only when the user intentionally overrides detection.
- Coding Plan Team seat missing: obtain or assign a seat before creating the
  team profile.
- Existing or conflicting profile: inspect `arkcli profile list --format json`
  before retrying with a different supported configuration.

`arkcli config init` is a compatibility path. New BytePlus profile workflows
should use `arkcli profile create`.
