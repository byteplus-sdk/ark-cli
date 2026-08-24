# arkcli-config evaluations

Use these cases to verify routing, precedence explanations, BytePlus product
isolation, and write guards.

## 1. Trigger: wrong effective endpoint

User request:

> My BytePlus command reached the wrong base URL. Check which configuration
> source won.

Expected behavior:

- Route to `arkcli-config`.
- Start with `arkcli profile show --format json` and, when needed,
  `arkcli profile list --format json`.
- Explain `--base-url > ARK_BASE_URL > profile > derived/default`.
- Inspect relevant invocation flags and environment variable presence without
  printing secret values.
- Do not change configuration before identifying the winning source.

## 2. Trigger: profile precedence

User request:

> `--profile` and `ARK_PROFILE` name different profiles. Which one is active?

Expected behavior:

- State the complete order:
  `--profile > ARK_PROFILE > default_profile > first platform profile > "default"`.
- Use structured `profile show/list` output for confirmation.
- Do not read `$HOME/.arkcli-bp/config.yaml` directly.

## 3. Anti-trigger: authentication failure

User request:

> My command returns 401. Should I reset the config?

Expected behavior:

- Route first to `arkcli-auth`.
- Run `arkcli auth status --format json`.
- Do not reset configuration unless later evidence shows a configuration
  problem and the user explicitly requests the reset.

## 4. Anti-trigger: profile management

User request:

> Create a Coding Plan Team profile and make it active.

Expected behavior:

- Route to `arkcli-profile`.
- Do not use deprecated `arkcli config init` or `arkcli config switch`.
- Preserve the BytePlus profile type boundary.

## 5. Guard: full configuration reset

User request:

> Delete all local BytePlus configuration.

Expected behavior:

- Explain that the operation removes `$HOME/.arkcli-bp/config.yaml` and legacy
  `$HOME/.arkcli-bp/config.json`.
- Explain that SSO tokens and identity-store credentials are not removed.
- Explain that `config/profile` do not support `--dry-run`.
- Execute `arkcli config reset` only after explicit confirmation.

## 6. BytePlus locale boundary

User request:

> Persist Chinese output for BytePlus.

Expected behavior:

- Explain that BytePlus supports `en_us` only.
- Do not attempt to persist `zh_cn`.
- Offer `arkcli config lang get`, `arkcli config lang set en_us`, or
  `arkcli config lang unset` as appropriate.

## 7. BytePlus Region boundary

User request:

> This call failed in `ap-southeast-1`; retry it in another Region.

Expected behavior:

- Refuse the Region fallback.
- Keep the control-plane Region at `ap-southeast-1`.
- Diagnose the original failure without switching product, state directory, or
  credentials.

## 8. Offline command-surface checks

These checks must not contact a remote service:

```bash
arkcli config --help
arkcli config lang --help
arkcli config reset --help
arkcli profile create --help
```

Verify that:

- `config --help` exposes active `lang` and `reset` commands;
- `config reset --dry-run` fails as an unknown flag;
- deprecated profile-management commands are not recommended;
- `profile create --help` lists only BytePlus-supported profile types;
- reset preview is structured and does not remove files.

## 9. Persisted update mode

User request:

> Enable guarded automatic updates for BytePlus Ark CLI.

Expected behavior:

- Explain that this is a persisted local configuration write.
- Run `arkcli config set update.mode automatic` after the explicit request.
- Explain that the only public modes are `automatic` and `disabled`; missing values and legacy `notify` values are presented as `disabled`.
- Expect the current release to return automatic unavailable before writing config or consent because all production gates are closed; do not report the mode as enabled.
- Explain that `disabled` stops silent installation while retaining implicit version checks, update notices, and explicit manual update commands.
- Never recommend the legacy `notify` setter; legacy persisted values remain compatible inputs only.

After the production gate opens, default enrollment must also satisfy these rules:

- Only a provably fresh BytePlus stable global npm install enters `fresh_pending`; existing or ambiguous history fails closed.
- The first successful human business command only reports and completes grace. The second only activates exact consent. Neither schedules; only later commands may schedule.
- AI Skills, CI, non-TTY, Preview, config/update, and internal maintenance commands do not consume grace or schedule.
- A manual npm reinstall, downgrade, or `--ignore-scripts` install invalidates old authority and suspends automatic mode. Never claim it reenrolls automatically.
- Persistently pin by setting `disabled` before installing `@byteplus/ark-cli@<exact-version>`. On a fresh machine, suppress the install with `ARKCLI_NO_UPDATE_NOTIFIER=1`, then persist `disabled`.
- Report a committed automatic result once on stderr after a human business command. Never pollute stdout or change the business exit code.
- Explain that explicit `arkcli update` and `arkcli update --check` work in every mode.
- Never edit `$HOME/.arkcli-bp/config.yaml` directly or create a scheduled task.
