---
name: arkcli-config
version: 0.3.0
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
| `arkcli config set update.mode automatic` | Explicitly enable automatic for this exact install; use it again to resume after a manual reinstall | Writes config and exact consent |
| `arkcli config set update.mode disabled` | Stop silent installation while retaining implicit checks, notices, and manual update commands | Writes `config.yaml` |
| `arkcli config reset` | Remove both local configuration files after an interactive confirmation | Destructive |

`arkcli config reset` does not remove BytePlus SSO tokens, the identity store,
or cached credentials. Use `arkcli auth logout` when the user explicitly wants
to remove authentication state.

The only public modes are `automatic` and `disabled`. `disabled` retains implicit version checks and update notices but never installs silently; explicit `arkcli update` and `arkcli update --check` also remain available. Missing mode values and legacy persisted `notify` values are presented as `disabled`. Legacy persisted `notify` values continue to read as `disabled`, but never recommend the legacy setter.

The fail-closed Windows, macOS, and Linux transactions and all BytePlus production gates are open. An npm postinstall never creates active mutation consent or updates immediately. Only a stable global npm install that proves `$HOME/.arkcli-bp` did not previously exist creates inert pending evidence for the exact install. First run and environment variables cannot bypass grace or exact consent.

### Fresh-install enrollment

Only a stable global npm install that can prove `$HOME/.arkcli-bp` did not previously exist defaults to `automatic`. Postinstall creates inert pending evidence bound to the exact install; it never creates active mutation consent.

- The first successful human business command reports automatic mode on stderr and completes grace. It does not schedule an update.
- The second successful human business command activates exact-install consent. It still does not schedule an update.
- Only the third and later human business commands may schedule an automatic patch update.
- Every CLI process started by npm `postinstall` (including its `+connect` invocation), and a user-run `+connect`, skips enrollment, implicit version checks, and automatic scheduling.
- AI Skills, CI, non-TTY invocations, Client Preview, `config`, `update`, and internal maintenance commands neither consume enrollment nor schedule automatic work.
- A committed update result appears once on stderr after the next successful human business command. It never changes stdout or the business exit code.

Production automatic updates deliberately keep the 24/48/72-hour and 10%/50%/100% cohort admission disabled. The original rollout implementation and regression coverage remain in place, and the online Probe still requires two independent observations. Normal automatic updates still require exact consent, a live registry target/SRI/tarball check, a one-use reservation, backoff, and every staged-apply safety boundary.

A manual npm reinstall, downgrade, changed install identity, or `--ignore-scripts` install suspends automatic mode when exact pending or consent evidence is absent. Never reuse authority from the previous install. The user can resume with `arkcli config set update.mode automatic`. For a persistent version pin, run:

```bash
arkcli config set update.mode disabled
npm i @byteplus/ark-cli@<exact-version> -g --registry https://registry.npmjs.org
```

On a fresh machine, set `ARKCLI_NO_UPDATE_NOTIFIER=1` on the historical-version install, then persist `disabled`. The policy lives in `$HOME/.arkcli-bp/config.yaml`, outside the npm package tree. `arkcli config reset` attempts to revoke exact consent before clearing configuration and never silently reenrolls the user.

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
- Use only the allowlisted `update.mode` key; never treat `config set` as a generic YAML-path writer. `disabled` still permits implicit checks and notices, and it does not disable explicit `arkcli update` or `arkcli update --check`.

## References

- [`references/arkcli-config-init.md`](references/arkcli-config-init.md) -
  migration from `config init` to BytePlus profile creation
- [`references/arkcli-config-profile.md`](references/arkcli-config-profile.md) -
  deprecated command mapping and active profile operations
- [`references/evals.md`](references/evals.md) - trigger, routing, precedence,
  and safety evaluations
