# Migrating from `config init`

`arkcli config init` is deprecated and hidden from normal help output. Create
BytePlus profiles with `arkcli profile create`.

## Product boundary

The standalone BytePlus binary fixes these values at build time:

| Setting | BytePlus value |
|---|---|
| Product identity | BytePlus |
| State directory | `$HOME/.arkcli-bp/` |
| Control-plane Region | `ap-southeast-1` |
| UI locale | `en_us` |

Do not copy a tenant selector, Region fallback, state path, or credentials from
another product.

## Supported profile types

BytePlus profile creation accepts:

- `platform`
- `coding-plan`
- `coding-plan-team`

Inspect the exact current flags before constructing a command:

```bash
arkcli profile create --help
```

Preview a pay-as-you-go profile:

```bash
arkcli profile create \
  --type platform \
  --region ap-southeast-1 \
  --project default \
  --set-default \
  --format json
```

After the user confirms the profile name, project, and default-profile change,
execute the command only after that confirmation.

For Coding Plan profiles, let Ark CLI detect the active subscription:

```bash
arkcli profile create --type coding-plan --set-default
arkcli profile create --type coding-plan-team --set-default
```

Do not force `--plan-tier` unless the user explicitly supplies the tier and
understands that it bypasses subscription detection. A team profile still
requires a valid assigned seat.

## Project resolution

Project Name participates in the effective runtime scope:

```text
active profile.project
  > active BytePlus identity fallback
  > default
```

There is no global `--project-name` or `ARK_PROJECT_NAME` runtime override.
The local `--project` flag on `profile create` persists the project in the new
profile. Use `arkcli profile project [<project-name>]` to change the persisted
project slice without adding a project override to a business command.

## Credentials

Do not place API keys directly into generated shell history unless the user
explicitly requests an inline key. Prefer the authenticated BytePlus identity
and profile key selection:

```bash
arkcli auth status --format json
arkcli profile keys list --format json
```

Never read or edit files under `$HOME/.arkcli-bp/` to change a project or API
key.
