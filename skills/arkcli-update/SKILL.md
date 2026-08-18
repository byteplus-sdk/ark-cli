---
name: arkcli-update
version: 1.0.0
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
# Default after a normal first npm install: guarded silent automatic updates
arkcli config set update.mode automatic

# Stop silent installation while retaining checks and update notices
arkcli config set update.mode notify

# Disable implicit checks, notices, applies, and postinstall cache warming
arkcli config set update.mode disabled
```

A normal npm postinstall initializes `automatic` only when `update.mode` has never been set and prints the opt-out command. Upgrade/reinstall must preserve an existing `notify` or `disabled`. If npm blocks postinstall, a stable npm-owned CLI initializes only after the first successful ordinary interactive command and prints the same opt-out; that command never updates immediately. CI, direct binaries, dev/candidate builds, non-TTY sessions, Preview, `config`/`update`, and internal commands remain `notify`. An Agent must never use the package default as permission to overwrite a persisted user choice.

In `automatic` mode, a normal interactive command first completes with the version that started it. The CLI then accepts only a cached official stable/latest target owned by the same npm global root. After the business process exits, the npm launcher validates a one-use nonce and strict handoff, then synchronously runs an external helper. The helper rereads `update.mode`, installs, and verifies both the package and executable; the next command uses the new version.

Only Windows currently enters automatic apply after its package/launcher rollback transaction passes every gate. macOS and Linux fail closed and retain explicit `arkcli update` until their own native recovery transactions are implemented and verified. Automatic apply is also refused in CI, non-TTY sessions, Client Preview, internal maintenance, `config`/`update` commands, candidate/integration builds, direct non-npm binaries, or npm-prefix mismatches. An apply failure preserves the completed business command's success and backs off the same target for 24 hours.

## Failure and platform routing

- Missing npm: relay the manual command; do not install Node or npm automatically.
- npm prefix / Node / NVM mismatch: stop and ask the user to activate the Node environment that installed this Ark CLI. Never update a different PATH prefix.
- Non-npm running binary: explain that the current copy is unaffected. A non-interactive retry needs explicit authorization before adding `--yes`; verified npm-package installation still does not mean that copy changed.
- Explicit `arkcli update` on Windows: the immediate result is only scheduled. An external helper applies the update after the parent exits and writes `update-apply.log`. Report completion only after a new `arkcli --version` shows the target or the log records `installed and verified`. The npm launcher synchronously waits for automatic apply on Windows; do not extend that safety claim to macOS or Linux while their automatic gate is closed.
- Windows apply uses a fail-closed rollback transaction. It preserves the previous package and npm-generated launchers, restores them after any npm/package/version/executable/launcher verification failure, and recovers an interrupted prior transaction before another mutation. Never interpret a failed apply as partial success, and never delete an `.arkcli-update-*` recovery directory during diagnosis.

## Prohibited behavior

- Do not impersonate npm postinstall or overwrite existing `notify/disabled`; an Agent changes policy only after an explicit user request.
- Do not simulate automatic updates with cron, a system service, or an Agent scheduled task.
- A version question never authorizes `arkcli update`.
- Never interrupt a normal business command or change its exit code because an update exists.
- Never interpret `status=unknown` as no update or up to date.

See [`references/evals.md`](references/evals.md) for evaluation cases.
