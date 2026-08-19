---
name: arkcli-update
version: 1.0.1
description: "Check or upgrade BytePlus Ark CLI and manage the persisted notify, automatic, and disabled update modes. Trigger for version checks, live refreshes, explicit upgrades, automatic-update opt-in/out, npm/Node/NVM prefix mismatches, or update-helper results."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli update --help"
---

# BytePlus Ark CLI version checks and updates

Read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) first and apply its caller attribution, structured-output, and confirmation rules. This command does not access a business account, so do not run `auth status` first.

## Check only

Use the zero-network local cache by default:

```bash
arkcli --format json update --check
```

Interpret `status`, not only the compatibility field `update_available`:

| `status` | Meaning | Agent action |
|---|---|---|
| `unknown` | No valid cache exists for this distribution | Report unknown; use `--refresh` only when a live answer is needed |
| `up_to_date` | Cache or registry confirms current is not older than latest | Report up to date and include `source` |
| `update_available` | `latest` is strictly newer than `current` | Report the version delta; do not update automatically |

`source` is `none`, `cache`, or `registry`; `checked_at` records the cache or query time. For a live registry query that also warms the cache:

```bash
arkcli --format json update --check --refresh
```

Preserve registry failures. Never present stale cache data as a live result.

## Apply an update

Enter the write flow only when the user explicitly asks to update the current Ark CLI. Show `current → latest` and the current distribution, then obtain confirmation in the current turn. Add `--yes` for a non-interactive invocation only after that confirmation:

```bash
arkcli update --yes
```

`update` launches npm and is classified as `opaque_external_execution`; it does **not** support `--dry-run`. Do not invent that flag or bypass the product command with a raw API.

Success requires npm to finish, the package under the same npm global root to equal `latest`, and the npm-managed target executable's `--version` to equal `latest`. Never report success after a verification failure.

Explicit `arkcli update` and `arkcli update --check` remain available in every mode. `disabled` affects only implicit behavior.

## Persisted update policy

Change policy only when the user explicitly requests it:

```bash
# Future explicit consent for this exact install after its product/platform gate opens.
# The current release returns automatic unavailable without writing config.
arkcli config set update.mode automatic

# Stop silent installation while retaining checks and update notices
arkcli config set update.mode notify

# Disable implicit checks, notices, applies, and postinstall cache warming
arkcli config set update.mode disabled
```

An npm postinstall, first run, or environment variable never infers automatic-update consent. A missing `update.mode` always resolves to the backwards-compatible `notify` default. The fail-closed Windows, macOS, and Linux transactions are implemented, but all six product/platform production gates are currently `false`; `config set update.mode automatic` therefore returns unavailable before reading or writing config or consent state. An Agent must never treat implementation readiness, a discovered release, or npm install shape as automatic-update authority.

After a product/platform gate passes real cross-version acceptance and is opened, `automatic` works as follows:

1. A normal interactive command completes with the version that started it.
2. The CLI requires the exact stable product/package/registry/tag, a forward patch, dual-clock observations, the 10/50/100 cohort, explicit consent, and a one-use rollout token.
3. A detached helper waits for the exact process to exit, downloads the reservation-pinned SRI/URL, and writes inert bytes into an isolated stage without executing npm, Node, lifecycle scripts, or the candidate.
4. Windows uses its persistent bootstrap, execution lease, journal, and FileId-bound no-replace renames; macOS uses `RENAME_SWAP`; Linux uses `RENAME_EXCHANGE`. Missing atomic-cutover support or any evidence drift is a safe miss.
5. The next command uses the new version only after post-cutover validation and consent rotation. Failure preserves the completed business command result and the prior installation or an exact rollback.

No product/platform currently enters production automatic apply. Even after a gate opens, CI, non-TTY sessions, Client Preview, AI Skill workflows, internal maintenance, `config`/`update` commands, candidate/integration builds, direct non-npm binaries, container/environment uncertainty, and npm-prefix mismatches remain safe misses. A failed target backs off for 24 hours.

## Failure and platform routing

- Missing npm: relay the manual command; do not install Node or npm automatically.
- npm prefix / Node / NVM mismatch: stop and ask the user to activate the Node environment that installed this Ark CLI. Never update a different PATH prefix.
- Non-npm running binary: explain that the current copy is unaffected. A non-interactive retry needs explicit authorization before adding `--yes`; verified npm-package installation still does not mean that copy changed.
- Explicit `arkcli update` on Windows: the immediate result is only scheduled. An external helper applies the update after the parent exits and writes `update-apply.log`. Report completion only after a new `arkcli --version` shows the target or the log records `installed and verified`.
- Windows apply uses a fail-closed rollback transaction. It preserves the previous package and npm-generated launchers, restores them after any npm/package/version/executable/launcher verification failure, and recovers an interrupted prior transaction before another mutation. Never interpret a failed apply as partial success, and never delete an `.arkcli-update-*` recovery directory during diagnosis.
- Explicit `arkcli update` on macOS/Linux retains the existing manual npm upgrade semantics. Do not claim that the still-closed automatic atomic-cutover path is already used by the explicit command.

## Prohibited behavior

- Do not impersonate npm postinstall or overwrite existing `notify/disabled`; an Agent changes policy only after an explicit user request.
- Do not simulate automatic updates with cron, a system service, or an Agent scheduled task.
- A version question never authorizes `arkcli update`.
- Never interrupt a normal business command or change its exit code because an update exists.
- Never interpret `status=unknown` as no update or up to date.

See [`references/evals.md`](references/evals.md) for evaluation cases.
