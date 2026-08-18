# Profile command migration and configuration scope

Profile operations belong to `arkcli profile`. The `arkcli config` group now
owns global UI language, the allowlisted `update.mode` policy, full
configuration reset, and compatibility for legacy scripts.

## Read-only inspection

```bash
# Show the effective active profile
arkcli profile show --format json

# Show one named profile
arkcli profile show <profile-name> --format json

# List all profiles
arkcli profile list --format json
```

Structured profile output masks secrets. Use it instead of reading
`$HOME/.arkcli-bp/config.yaml`.

## Profile writes

```bash
# Change the default profile
arkcli profile use <profile-name> --format json

# Preview creation
arkcli profile create --type platform --format json

# Delete after confirming the exact target
arkcli profile delete <profile-name> --format json

# Re-select the project after reviewing the replacement scope
arkcli profile project <project-name>
```

`profile use` changes the top-level default-profile pointer. `profile delete`
removes one local profile. `profile project` can replace project-scoped
profiles, so inspect its output and confirm before using `--yes`.

Use [`../SKILL.md`](../SKILL.md) for configuration precedence and
[`../../arkcli-profile/SKILL.md`](../../arkcli-profile/SKILL.md) for the full
profile workflow.

## Active `config` commands

```bash
arkcli config lang get
arkcli config lang set en_us
arkcli config lang unset
arkcli config set update.mode notify
arkcli config set update.mode automatic
arkcli config set update.mode disabled
arkcli config reset
```

BytePlus rejects `zh_cn`. `config reset` removes
`$HOME/.arkcli-bp/config.yaml` and legacy
`$HOME/.arkcli-bp/config.json`; it does not remove the BytePlus identity store
or SSO tokens.

## Deprecated mapping

| Deprecated command | Active replacement |
|---|---|
| `arkcli config init` | `arkcli profile create` |
| `arkcli config list` | `arkcli profile list` |
| `arkcli config show` | `arkcli profile show` |
| `arkcli config switch <name>` | `arkcli profile use <name>` |
| `arkcli config delete <name>` | `arkcli profile delete <name>` |

The deprecated commands may still execute for compatibility, but they are
hidden from normal help and must not be used in new automation.
