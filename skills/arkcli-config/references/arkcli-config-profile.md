# Profile command migration and configuration scope

Profile operations belong to `arkcli profile`. The `arkcli config` group now
owns global UI language, the allowlisted `update.mode` policy, full
configuration reset, and compatibility for legacy scripts.

## Routine admission

Use `arkcli auth status --format json` and `arkcli auth whoami --format json`
for the masked current identity/Profile. Use `arkcli resources list --modality
text --format json` for the current chat default and follow the Resources Skill
for compatibility checks. These summaries do not prove a data-plane request
will succeed; report missing evidence instead of inferring key validity.

## Explicit Profile inspection and management

`profile show/list/keys list` may synchronize remote keys and write back the
local key inventory, clear an invalid key, or select another default key.
Explain that impact before using them for an explicit Profile task. Do not use
them for routine Chat/Gen admission or a no-configuration/key-change request,
and do not bypass that boundary with deprecated `config show/list`.

```bash
# Show the effective active profile
arkcli profile show --format json

# Show one named profile
arkcli profile show <profile-name> --format json

# List all profiles
arkcli profile list --format json
```

Structured profile output masks secrets but is not side-effect-free. Do not read
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
arkcli config set update.mode automatic
arkcli config set update.mode disabled
arkcli config reset
```

The only public update modes are `automatic` and `disabled`. `disabled` stops
silent installation while retaining implicit version checks, notices, and
explicit `arkcli update` / `arkcli update --check`. Legacy `notify` values in
persisted configuration remain compatible inputs but are not settable.

With the production gates open, a provably fresh BytePlus stable global npm
install whose version equals registry `latest` starts inert enrollment bound to
the exact install. The first successful eligible human business command
completes grace and consent; after active revalidation it may schedule. A
current-latest reinstall preserves the mode and reissues exact consent only for
existing `automatic`; a non-latest install persists `disabled`.

For a persistent version pin, set `disabled` before installing the exact
`@byteplus/ark-cli` version. Installing `@latest` does not turn `disabled` back
into `automatic`; recover by installing latest and then explicitly setting
automatic. The policy lives in `$HOME/.arkcli-bp/config.yaml`, outside the npm
package tree.

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
