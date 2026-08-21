# Version checks and explicit upgrades

This reference only answers whether the installed BytePlus Ark CLI is current
and applies an upgrade that the user explicitly requested. It does not access a
business account, so do not run `auth status` first.

The reading and writing of `update.mode`, its default, and automatic production
gates belong to
[`../../arkcli-config/SKILL.md`](../../arkcli-config/SKILL.md). Never infer
automatic-update consent from a discovered release, an ordinary npm install,
or postinstall.

## Check only

Read the zero-network local cache first:

```bash
ARKCLI_NO_UPDATE_NOTIFIER=1 \
ARKCLI_CALLER_TYPE=ai_agent \
ARKCLI_CALLER_NAME=<agent-id> \
ARKCLI_SKILL_NAME=arkcli-shared \
arkcli --format json update --check
```

Interpret `status`, not only the compatibility field `update_available`:

| `status` | Meaning | Agent action |
|---|---|---|
| `unknown` | No valid cache exists for this distribution | Report unknown; use `--refresh` only when a live answer is needed |
| `up_to_date` | Cache or registry confirms current is not older than latest | Report up to date and include `source` |
| `update_available` | `latest` is strictly newer than `current` | Report the version delta; do not update automatically |

`source` is `none`, `cache`, or `registry`; `checked_at` records the cache or
query time. Refresh only when the user needs a live answer:

```bash
ARKCLI_NO_UPDATE_NOTIFIER=1 \
ARKCLI_CALLER_TYPE=ai_agent \
ARKCLI_CALLER_NAME=<agent-id> \
ARKCLI_SKILL_NAME=arkcli-shared \
arkcli --format json update --check --refresh
```

`--refresh` queries the npm registry embedded in the current binary. Preserve
registry failures. Never present stale cache data as a live result, and never
interpret `status=unknown` as no update or up to date.

## Explicit upgrade

Enter the write flow only when the user explicitly asks to upgrade in the
current turn. Check first, show `current -> latest` and the distribution, then
obtain confirmation. Add `--yes` to a non-interactive invocation only after
that confirmation:

```bash
ARKCLI_NO_UPDATE_NOTIFIER=1 \
ARKCLI_CALLER_TYPE=ai_agent \
ARKCLI_CALLER_NAME=<agent-id> \
ARKCLI_SKILL_NAME=arkcli-shared \
arkcli update --yes
```

Explicit `arkcli update` and `arkcli update --check` remain available in every
mode. `disabled` affects only implicit behavior.

`update` launches npm and is classified as `opaque_external_execution`; it
does not support `--dry-run`. Do not invent Client Preview or bypass the
product command with a raw API. Use the version delta and explicit confirmation
as the safety boundary.

Report success only when npm finishes, the package under the same npm global
root equals `latest`, and the npm-managed target executable's `--version` also
equals `latest`. Never report success after a command or verification failure.

## Failure and platform routing

- Missing npm: relay the CLI's manual command; do not install Node or npm.
- npm prefix, Node, or NVM mismatch: stop and ask the user to activate the Node
  environment that installed this Ark CLI. Never update a different PATH
  prefix.
- Non-npm running binary: explain that the current copy is unaffected. A
  verified npm package still does not mean that copy changed.
- On Windows, the immediate explicit-update result is only scheduled. Report
  completion only after a new `arkcli --version` shows the target or
  `update-apply.log` records `installed and verified`.
- Preserve the previous installation, logs, and `.arkcli-update-*` recovery
  directories after a Windows failure. Never describe failure as partial
  success.
- On macOS and Linux, explicit update retains manual npm-upgrade semantics. Do
  not describe the automatic atomic-cutover path as part of the explicit
  command.

Do not simulate automatic updates with cron, a system service, or an Agent
scheduled task. Never interrupt a normal business command or change its exit
code because an update exists.
