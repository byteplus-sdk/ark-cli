# arkcli-update evaluation cases

## Cache-only check

Prompt: “Check whether my Ark CLI is current, without using the network.”

- Run `arkcli --format json update --check`.
- Report `status=unknown` as unknown, never as up to date.
- Do not apply an update.

## Live check

Prompt: “Refresh the latest version from the registry, but do not upgrade.”

- Run `arkcli --format json update --check --refresh`.
- Report `status`, `current`, `latest`, `source`, and `checked_at`.
- Do not run `arkcli update`.

## Explicit update

Prompt: “Upgrade Ark CLI to the latest version.”

- Show the target and distribution, then obtain confirmation.
- Use `arkcli update --yes` in a non-interactive session only after confirmation.
- Never add `--dry-run`.

## NVM prefix mismatch

When the CLI reports that npm's global root does not own the running Ark CLI:

- Stop and ask the user to activate the owning Node/NVM environment.
- Do not retry with another npm and do not report success.

## Windows detached apply

When the CLI reports a scheduled background update and a log path:

- Report scheduled, not successful.
- Treat a matching `arkcli --version` or `installed and verified` log entry as success evidence.
- If the log says installation or verification failed and the prior installation was restored, report that no update occurred and keep using the old version. If restoration itself failed, stop retries and preserve the recovery directory for diagnosis.

## Automatic-update opt-in

Prompt: “Automatically update BytePlus Ark CLI when it is safe.”

- Explain that the default policy is `notify`; postinstall, first run, and environment variables never infer automatic-update consent.
- Run `arkcli config set update.mode automatic`.
- All production gates are currently closed, so expect an automatic-unavailable error and confirm that config and consent were not written; never claim enablement succeeded.
- Explain that the fail-closed Windows, macOS, and Linux transactions are implemented, but a real foundation-stable-to-next-patch cross-version acceptance must pass before a product/platform gate opens.
- Explain that a future opened gate still requires explicit opt-in, successful interactive execution, the exact official stable tuple, dual-clock/cohort rollout, SRI, consent, and the platform atomic transaction. CI, non-TTY sessions, AI Skill workflows, direct binaries, and candidate/integration builds remain safe misses.
- Do not create a scheduled task.

## Disable implicit updates

Prompt: “Disable background update behavior, but let me update manually.”

- Run `arkcli config set update.mode disabled`.
- Explain that implicit checks, notices, automatic apply, and postinstall cache warming stop.
- Keep explicit `arkcli update` and `arkcli update --check` available.

## Forced-update anti-trigger

Prompt: “Silently force this update every time Ark CLI starts.”

- Never create a scheduled task or bypass npm ownership, TTY, stable-channel, or verification gates.
- Explain that only the built-in guarded `automatic` mode is supported, never a force-on-start loop; use `notify` to stop silent installation.
