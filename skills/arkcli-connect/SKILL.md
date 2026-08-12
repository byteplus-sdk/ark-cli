---
name: arkcli-connect
version: 0.4.0
description: "Install, inspect, refresh, or uninstall BytePlus Ark CLI Skills for supported local AI agents or one explicit Skills directory. Use for managed Skill updates, project-scoped installation, automatic public-package post-install setup, remote or embedded Skill refresh, and explicit legacy prefix cleanup."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli +connect --help"
---

# Connect BytePlus Ark CLI Skills

Before using this Skill, read
[`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) for common safety and
product-isolation rules.

`arkcli +connect` copies the Skills available to the current BytePlus build into
supported local AI agent directories. Internal builds may embed Skills. Public
npm builds download their reviewed English Skills from the package manifest,
with CDN as primary and GitHub Release as fallback. Installation is local and
does not require BytePlus authentication.

## When To Trigger

- Install BytePlus Ark CLI Skills into detected local AI agents.
- Install them into a project-level or custom Skills directory.
- List supported agents and local detection status.
- Preview or remove installed Ark CLI Skills.
- Explain why package installation automatically connected Skills.
- Explain `--refresh` or `--keep-conflicting`.

## When NOT To Trigger

- `401`, login, token, or permission failures: use
  [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).
- Profile, Region, project, base URL, or API key selection: use
  [`../arkcli-config/SKILL.md`](../arkcli-config/SKILL.md) or
  [`../arkcli-profile/SKILL.md`](../arkcli-profile/SKILL.md).
- Model invocation or code generation: use the owning business Skill.
- Agent runtime, model-provider, or MCP configuration: `+connect` installs
  Skills only and does not configure those systems.

## Command surface

| Command | Behavior |
|---|---|
| `arkcli +connect list` | List supported agents, paths, and detection status; read-only |
| `arkcli +connect` | Install into all detected agent Skill directories |
| `arkcli +connect --path <skills-dir>` | Install only into one explicit Skills directory |
| `arkcli +connect uninstall` | Remove unchanged Skills recorded in the ArkCLI ownership manifest from all detected agents |
| `arkcli +connect uninstall --path <skills-dir>` | Remove unchanged managed Skills only from one explicit directory |
| `arkcli +connect uninstall --purge-prefix` | Destructively remove every `ark-` or `arkcli-` directory or symlink from detected targets |
| `arkcli +connect uninstall --path <skills-dir> --purge-prefix` | Apply destructive prefix cleanup only to one explicit directory |

Installation is the default action. The only subcommands are `list` and
`uninstall`. Do not invent `install`, `setup`, `sync`, or `remove`
subcommands. `--path` is a flag, not a subcommand.

## Agent execution order

1. If the user only asks what is supported, run `arkcli +connect list` and
   stop.
2. `+connect` does not support `--dry-run`; inspect targets with `list`.
3. Explain that installation makes the current catalog authoritative, while
   normal uninstall removes only unchanged ownership-manifest entries.
4. Require an additional explicit confirmation before any `--purge-prefix`
   command because it includes user-managed entries.
5. Run the exact confirmed command.
6. Report installed, purged, skipped, or removed counts and paths.

## Managed ownership contract

Each target contains `.arkcli-managed-skills.json`, which records the exact
Skill names, product, and tree digests written by ArkCLI.

Normal installation:

- overwrites every exact current-catalog name, regardless of previous owner,
  digest, product, or local edits;
- removes every previously manifest-owned name that is absent from the current
  catalog, even when its bytes changed;
- preserves every different name that is neither in the current catalog nor
  in the previous manifest;
- stages, backs up, installs, and updates the manifest as one rollback-safe
  transaction.

The manifest digests remain provenance and protect normal uninstall. They no
longer block installation. Local customization under an official catalog name
is intentionally ephemeral; use a different Skill name to preserve it.

The historical `arkcli-managed-agent` alias is retired through this general
rule when an old manifest owns it, while the canonical `arkcli-agent` is
installed from the current catalog. There is no BytePlus-only install bypass.

Use `uninstall --purge-prefix` only for an explicitly confirmed legacy cleanup.
That mode restores the old broad prefix deletion and can remove user content.

## `--path` isolation

```bash
arkcli +connect --path .claude/skills
```

- The value is the exact Skills directory, not a project root.
- Relative paths resolve from the current working directory.
- Absolute paths are used unchanged.
- Only that directory is created and its current catalog plus previous
  manifest are converged.
- Other different-name entries remain untouched.
- The command does not scan global agents, clean legacy private agent
  directories, or remove external conflicting Skills outside the target.

## Detected-agent installation

Default installation:

- detects supported agents from local filesystem paths;
- deduplicates agents that share one Skills directory;
- prefers an agent-specific detection home, such as Codex or Pi, when choosing
  the representative name for a shared target;
- installs each unique target path once;
- cleans legacy private Ark CLI copies that would otherwise create duplicate
  Skill names;
- may remove specifically recognized external generation Skills that conflict
  with Ark CLI routing.

Use `arkcli +connect --keep-conflicting` only when the user explicitly wants to
preserve those recognized external Skills. This flag applies to detected-agent
installation; isolated `--path` installation does not remove them.

Do not hardcode the supported-agent list in an answer. Use:

```bash
arkcli +connect list
```

## BytePlus `--refresh`

Delivery mode depends on the build:

- public npm BytePlus: refresh the package-local Skill tree from CDN primary,
  then GitHub Release fallback when configured;
- internal BytePlus: report embedded mode and continue with the embedded tree.

Do not guess. Report the source printed by the CLI, such as `cdn`, `github`, or
`embedded`, and never claim a successful download for an embedded build.

## Uninstall guard

Normal `uninstall` removes only manifest-owned, digest-unchanged Skills. It
preserves unowned and user-modified entries. `--purge-prefix` is the separate
destructive compatibility path.

Always:

1. run `arkcli +connect list` for global scope or resolve the exact `--path`;
2. inspect the exact target with `list`;
3. show the affected paths and whether managed-only or prefix purge applies;
4. obtain explicit confirmation for prefix purge;
5. execute the exact confirmed command.

## Automatic package setup

The package post-install script can run `+connect` automatically when it has
an interactive controlling terminal. It silently skips CI, non-interactive
sessions, unsupported platforms, or missing binaries. Set
`ARKCLI_SKIP_POSTINSTALL=1` before package installation to disable automatic
connection.

Automatic post-install always uses the same authoritative-catalog transaction
and never enables `--purge-prefix`. It is non-blocking for package installation.
On success, its compact summary reports the unique target count and the
representative agent names without per-target path progress. A filesystem or
connection failure does not imply a BytePlus authentication failure.

## Guard Checklist

- Use `list` for read-only discovery.
- Never generate `--dry-run` for this domain; use `list` and explicit confirmation.
- Overwrite every exact official catalog name during install; preserve only
  unrelated different-name entries.
- Preserve modified manifest-owned entries during normal uninstall.
- Treat `--purge-prefix` as an explicit destructive compatibility operation.
- Confirm global versus explicit `--path` scope.
- Never translate `--path` into an agent selector.
- Do not invent subcommands.
- Do not require login.
- Distinguish public remote refresh from internal embedded mode using actual CLI output.
- Do not use `+connect` to diagnose a business-command `401`.

## References

- [`references/arkcli-connect.md`](references/arkcli-connect.md) - detailed
  paths, cleanup, post-install, and error behavior
- [`references/evals.md`](references/evals.md) - routing, preview, uninstall,
  and anti-hallucination evaluations
