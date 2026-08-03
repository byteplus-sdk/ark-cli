# `+connect` detailed reference

Read [`../SKILL.md`](../SKILL.md) first.

## Exact command grammar

```text
arkcli +connect [flags]
arkcli +connect list
arkcli +connect uninstall [flags]
```

Valid parent flags:

- `--path <skills-dir>`
- `--refresh`
- `--keep-conflicting`

Valid uninstall flags:

- `--path <skills-dir>`
- `--purge-prefix`

There is no `install`, `setup`, `sync`, or `remove` subcommand.

## Default installation

```bash
arkcli +connect
```

The default workflow:

1. detects supported agents from local directories;
2. maps each detected agent to its Skills directory;
3. deduplicates shared directory paths;
4. reads the Ark CLI ownership manifest and stages the complete current catalog;
5. removes recognized routing conflicts unless `--keep-conflicting` is set;
6. atomically overwrites every current-catalog name and retires names that left
   the catalog;
7. cleans duplicate legacy private copies where required;
8. reports the result for each unique target path.

Detection and installation paths may differ for agents that consume a shared
Skills directory. Always use `arkcli +connect list` instead of assuming a
path.

## Explicit directory mode

```bash
cd /path/to/project
arkcli +connect --path .claude/skills
```

The result is similar to:

```text
/path/to/project/.claude/skills/arkcli-auth/SKILL.md
/path/to/project/.claude/skills/arkcli-chat/SKILL.md
/path/to/project/.claude/skills/arkcli-shared/SKILL.md
/path/to/project/.claude/skills/.arkcli-managed-skills.json
```

Explicit directory mode:

- resolves a relative value from the current working directory;
- accepts an absolute value unchanged;
- creates the target directory when installing;
- overwrites every exact current-catalog name inside that directory;
- retires previous manifest names that left the catalog;
- preserves unrelated different-name `ark-` and `arkcli-` directories;
- does not scan global agents;
- does not clean legacy private directories;
- does not remove external conflicts outside the directory;
- does not change agent runtime or MCP configuration.

An empty `--path` is rejected and must never fall back to global installation
or uninstallation.

## Ownership and cleanup details

Each target contains `.arkcli-managed-skills.json`. The manifest records the
exact managed Skill names and their full-tree SHA-256 digests.

Install and uninstall intentionally use different digest policies:

- install treats the current CLI catalog as authoritative and overwrites every
  exact current-catalog name, regardless of ownership or local changes;
- install retires every previous manifest name absent from the current catalog,
  regardless of local changes;
- install preserves a different name that is in neither the current catalog nor
  the previous manifest, even when it begins with `ark-` or `arkcli-`;
- install stages the full incoming tree before moving current paths to backup;
  any failure rolls the whole catalog and manifest back;
- normal uninstall still requires the current digest to match the recorded
  digest and preserves modified managed entries.

`arkcli-managed-agent` to `arkcli-agent` follows the same catalog transaction:
an old manifest-owned alias leaves the catalog and is retired, while the
canonical current name is overwritten with the selected bundle. The outcome
does not depend on the old manifest product or digest.

For intentional legacy cleanup, use:

```bash
arkcli +connect uninstall --purge-prefix
arkcli +connect uninstall --path /exact/skills/dir --purge-prefix
```

`--purge-prefix` is destructive: it removes every directory or symlink whose
name begins with `ark-` or `arkcli-`, including user-managed entries. The flag
is the explicit opt-in to that broad cleanup. Symlinks are unlinked without
following and deleting their source target.

## Safety boundary

The entire `+connect` domain rejects `--dry-run`. Use `+connect list` to
inspect detected targets. Restate the exact path and broad cleanup impact
before any `uninstall --purge-prefix` command.

## BytePlus refresh behavior

Public BytePlus npm packages use the reviewed English Skill manifest published
with the release. `arkcli +connect --refresh` prefers the BytePlus CDN bundle
and falls back to the BytePlus GitHub release bundle. Internal BytePlus builds
may use the English bundle embedded in the current binary.

The CLI output is authoritative: report the actual selected source as `cdn`,
`github`, or `embedded`. Do not claim a remote refresh when the CLI reports
`embedded`.

## Post-install behavior

The package post-install script:

1. skips when `ARKCLI_SKIP_POSTINSTALL=1` or a CI environment is detected;
2. validates the platform binary;
3. attempts to open an interactive controlling terminal;
4. runs the binary with `+connect` when that terminal is available;
5. treats connection failure as non-blocking for package installation.

Piped and other non-interactive installations skip automatically. The
post-install path uses the same authoritative-catalog transaction and never
opts into `--purge-prefix`.

## Common messages and errors

| Message or error | Meaning | Recovery |
|---|---|---|
| `No AI agents detected` | No supported local agent directory was found | Install an agent or use an explicit `--path` |
| `No skills found locally` | No valid local or refreshed Skill bundle is available | Install a complete package or retry refresh when the configured source is available |
| `managed skill was modified` during uninstall | A previously managed Skill no longer matches its recorded digest | Normal uninstall preserves it; use a separately confirmed `--purge-prefix` only when broad deletion is intended |
| `--path must not be empty` | An explicit path flag had no value | Supply the exact Skills directory |
| `permission denied` | The target directory is not writable | Fix permissions or choose an authorized directory |
| `copy ...: not a directory` | A regular file blocks a Skill directory name | Move the blocking file, then retry |

Do not route these local filesystem failures to BytePlus authentication.

`arkcli-managed-agent` is not a current installable Skill name. If an old
manifest owns it, the next install retires it under the general catalog rule.
Current BytePlus embedded, CDN, and GitHub bundles publish only `arkcli-agent`.
