---
name: arkcli-config
version: 0.2.0
description: "Diagnose BytePlus Ark CLI configuration precedence, effective profile, Region, project, base URL, API key, environment, UI language, persisted update mode, and local reset behavior. Use for unclear overrides, update opt-in/out, legacy config migration, inspection, or reset."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli config --help"
---

# BytePlus Ark CLI configuration

Before using this Skill, read
[`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md). It defines the
BytePlus product boundary, authentication flow, output rules, and write
confirmation requirements.

Use this Skill for configuration diagnosis and the remaining `arkcli config`
commands. Use [`../arkcli-profile/SKILL.md`](../arkcli-profile/SKILL.md) for
profile creation, selection, deletion, project changes, resource defaults, and
API key selection.

## BytePlus configuration boundary

- BytePlus state is stored under `$HOME/.arkcli-bp/`.
- The control-plane Region is fixed to `ap-southeast-1`. Do not fall back to
  another Region or product.
- BytePlus supports the `en_us` UI locale only.
- Do not read or edit files under `$HOME/.arkcli-bp/` directly. Use Ark CLI
  commands so secrets remain masked and migrations remain intact.

## When To Trigger

- A command used the wrong profile, account, project, Region, base URL, or API
  key.
- The user needs to understand which flag, environment variable, profile, or
  identity value won.
- The user asks about `ARK_PROFILE`, `ARK_API_KEY`, `ARK_BASE_URL`, or
  `ARK_CLIENT_ENV`, including whether legacy `ARK_REGION` or
  `ARK_PROJECT_NAME` values still have an effect.
- The user needs `arkcli config lang`, `arkcli config reset`, or a migration
  from a deprecated `arkcli config` profile command.
- The user explicitly wants to enable, disable, or restore automatic-update behavior.

## When NOT To Trigger

- Missing login, an expired session, `401`, or permission failure: use
  [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).
- Profile creation, switching, deletion, project selection, or API key
  management: use [`../arkcli-profile/SKILL.md`](../arkcli-profile/SKILL.md).
- A business task such as model discovery, generation, deployment, or usage:
  diagnose configuration only if evidence points here, then return to the
  owning business Skill.

## Agent execution order

1. Start with structured, read-only commands:

   ```bash
   arkcli profile show --format json
   arkcli profile list --format json
   ```

2. If the effective identity is still unclear, inspect the masked
   authentication summary:

   ```bash
   arkcli auth status --format json
   ```

3. Identify invocation-scoped flags and relevant environment variables without
   printing secret values.
4. Explain the winning source explicitly before proposing a change.
5. After the configuration issue is resolved, return to the user's original
   task.

## Resolution precedence

Profile selection, from highest to lowest:

```text
--profile
  > ARK_PROFILE
  > config.yaml default_profile
  > first supported platform profile by name
  > "default"
```

Other effective values follow these product rules:

| Value | Highest to lowest |
|---|---|
| API key | `--api-key` > `ARK_API_KEY` > active profile > bound identity; legacy fallback is used only when no identity is bound |
| Base URL | `--base-url` > `ARK_BASE_URL` > active profile value > value derived from profile type and Region > BytePlus default |
| Region | active profile > `ap-southeast-1`; BytePlus rejects any other persisted effective Region |
| Project | active profile project > active identity fallback > `default` |
| Environment | `--env` > `ARK_CLIENT_ENV` > active profile > `prod` |

Do not infer product identity from these values. Product identity is compiled
into the BytePlus binary.

`--region`, `--project-name`, `ARK_REGION`, and `ARK_PROJECT_NAME` do not
override runtime scope. Select or update a profile instead. For temporary
data-plane calls, `--api-key`/`ARK_API_KEY` and
`--base-url`/`ARK_BASE_URL` are the supported override pair; an explicit base
URL must never reuse a profile API key.

## Active commands

| Command | Purpose | Safety |
|---|---|---|
| `arkcli config lang get` | Show the persisted effective UI locale | Read-only |
| `arkcli config lang set en_us` | Persist the only locale supported by BytePlus | Writes `config.yaml` |
| `arkcli config lang unset` | Remove the persisted locale and use the BytePlus build default | Writes `config.yaml` |
| `arkcli config set update.mode notify` | Stop silent installation while retaining implicit checks and notices | Writes `config.yaml` |
| `arkcli config set update.mode automatic` | Guarded updates; initialized by default after npm install, with automatic apply currently open only on Windows | Writes `config.yaml` |
| `arkcli config set update.mode disabled` | Disable implicit update behavior, while preserving manual update commands | Writes `config.yaml` |
| `arkcli config reset` | Remove both local configuration files after an interactive confirmation | Destructive |

`arkcli config reset` does not remove BytePlus SSO tokens, the identity store,
or cached credentials. Use `arkcli auth logout` when the user explicitly wants
to remove authentication state.

A normal npm postinstall initializes `automatic` only when `update.mode` has never been set and prints `arkcli config set update.mode notify` as the opt-out. Upgrade/reinstall preserves existing `notify/disabled`. If npm blocks postinstall, only a stable npm-owned CLI initializes after its first successful ordinary interactive command and prints the same opt-out; that command never updates immediately. Other skipped cases remain non-automatic.

BytePlus rejects `zh_cn` even though the shared command parser recognizes that
locale for other products.

## Deprecated compatibility

The following commands remain only for old scripts and are hidden from normal
`arkcli config --help` output:

| Deprecated | Use instead |
|---|---|
| `arkcli config init` | `arkcli profile create` |
| `arkcli config list` | `arkcli profile list` |
| `arkcli config show` | `arkcli profile show` |
| `arkcli config switch <name>` | `arkcli profile use <name>` |
| `arkcli config delete <name>` | `arkcli profile delete <name>` |

Do not generate new automation with deprecated commands.

## Guard Checklist

- Diagnose with `profile show/list` before changing anything.
- Never echo a full token, API key, access key, or secret key.
- Confirm the exact target before `profile use`, `profile delete`, project
  changes, language changes, or `config reset`.
- The entire `config` and `profile` domains reject `--dry-run`; inspect with
  `show/list`, restate the exact mutation, and obtain confirmation instead.
- Keep BytePlus on `ap-southeast-1`; do not retry through another product.
- Return to the user's original business task after configuration is fixed.
- Use only the allowlisted `update.mode` key; never treat `config set` as a generic YAML-path writer. `disabled` does not disable explicit `arkcli update` or `arkcli update --check`.

## References

- [`references/arkcli-config-init.md`](references/arkcli-config-init.md) -
  migration from `config init` to BytePlus profile creation
- [`references/arkcli-config-profile.md`](references/arkcli-config-profile.md) -
  deprecated command mapping and active profile operations
- [`references/evals.md`](references/evals.md) - trigger, routing, precedence,
  and safety evaluations
