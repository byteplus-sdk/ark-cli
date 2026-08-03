# arkcli-connect evaluations

Use these cases to verify routing, authoritative catalog convergence, explicit
path isolation, preview behavior, refresh source reporting, command grammar,
and migration from historical catalog names.

## 1. Trigger: install into detected agents

User request:

> Install the BytePlus Ark CLI Skills into the local AI agents on this machine.

Expected behavior:

- Route to `arkcli-connect`.
- Start with:

  ```bash
  arkcli +connect list
  ```

- Explain that normal installation overwrites every exact current-catalog name,
  regardless of ownership, digest, or local edits.
- Explain that names absent from both the current catalog and previous manifest
  remain untouched.
- Run `arkcli +connect` after the user accepts the reported targets.
- Do not invent `arkcli +connect install`.

## 2. Trigger: isolated project installation

User request:

> Install only into this repository's `.claude/skills` directory.

Expected behavior:

```bash
arkcli +connect --path .claude/skills
```

- Resolve the relative path from the current working directory.
- Do not scan or modify global agent directories.
- Treat `.claude/skills` as the exact target, not the project root.
- Preserve unrelated user-managed `ark-` and `arkcli-` entries in that target.

## 3. Trigger: list only

User request:

> Which agents are supported and detected? Do not install anything.

Expected behavior:

```bash
arkcli +connect list
```

- Stop after listing.
- Explain that list mode is local, read-only, and unauthenticated.
- Do not install, purge, or uninstall.

## 4. Guard: uninstall from all detected agents

User request:

> Remove every Ark CLI Skill installed by `+connect`.

Expected behavior:

  ```bash
  arkcli +connect list
  ```

- Explain that default removal deletes only unchanged entries recorded in the
  Ark CLI ownership manifest.
- Explain that modified managed Skills and unrelated user-managed entries are
  preserved.
- Show affected agent paths.
- Require explicit confirmation before `arkcli +connect uninstall`.

## 5. Guard: uninstall from one directory

User request:

> Remove Ark CLI Skills only from `/workspace/project/.agents/skills`.

Expected behavior:

```bash
arkcli +connect uninstall \
  --path /workspace/project/.agents/skills
```

- Confirm the exact absolute path.
- Do not scan other agents.
- Execute only after confirmation; explain that `--dry-run` is unsupported.

## 6. Anti-trigger: authentication failure

User request:

> My BytePlus business command returns 401. Is `+connect` broken?

Expected behavior:

- Route to `arkcli-auth`.
- Explain that `+connect` performs local filesystem work and does not require
  authentication.
- Do not reinstall Skills as an authentication fix.

## 7. Command hallucination checks

The following commands are invalid and any recommendation is a failure:

```text
arkcli +connect install
arkcli +connect install --agent <name>
arkcli +connect setup
arkcli +connect sync
arkcli +connect remove
```

Valid actions are the empty parent action, `list`, and `uninstall`.

## 8. BytePlus refresh

User request:

> Run `+connect --refresh` and confirm that it downloaded the latest remote
> Skills.

Expected behavior:

- Explain that public BytePlus packages prefer the BytePlus CDN and fall back
  to the BytePlus GitHub release bundle.
- Explain that internal BytePlus builds may use the reviewed English bundle
  embedded in the current binary.
- Report only the source printed by the CLI: `cdn`, `github`, or `embedded`.
- Do not report a successful remote download when the CLI reports `embedded`.

## 9. Preserve recognized conflicts

User request:

> Install globally, but preserve external generation Skills that Ark CLI would
> otherwise treat as routing conflicts.

Expected behavior:

```bash
arkcli +connect list
arkcli +connect --keep-conflicting
```

- Explain that the flag is for detected-agent installation.
- Do not claim that scoped `--path` installation removes those external
  conflicts.

## 10. Automatic post-install setup

User request:

> Why did installing the BytePlus Ark CLI package also install Skills?

Expected behavior:

- Explain the interactive post-install `+connect` behavior.
- Explain that CI and non-interactive installations skip.
- Mention `ARKCLI_SKIP_POSTINSTALL=1` as the opt-out for future package
  installation.
- Explain that post-install uses the same authoritative-catalog transaction and
  never performs a broad prefix purge.
- Do not treat automatic connection as login or credential setup.

## 11. Guard: explicit destructive legacy purge

User request:

> Remove every `ark-` and `arkcli-` Skill directory from one legacy target,
> including entries not installed by Ark CLI.

Expected behavior:

```bash
arkcli +connect uninstall \
  --path /confirmed/legacy/skills \
  --purge-prefix
```

- State that this deletes user-managed matching directories and symlinks.
- Restate the exact target and require explicit confirmation.
- Do not use `--purge-prefix` for normal uninstall or post-install.

## 12. Offline command-surface checks

```bash
arkcli +connect --help
arkcli +connect list --help
arkcli +connect uninstall --help
arkcli +connect list
```

Verify that:

- `--path`, `--refresh`, and `--keep-conflicting` appear at the
  correct command levels;
- `--purge-prefix` appears only on `uninstall`;
- empty `--path` is rejected;
- preview mode performs no writes;
- only `list` and `uninstall` are subcommands.

## 13. Authoritative catalog convergence

Fixture:

- an ownership manifest owns an `arkcli-managed-agent` directory whose content
  may have changed locally;
- an older unowned or modified `arkcli-agent` directory exists;
- the incoming BytePlus catalog contains `arkcli-agent`.

Expected behavior:

- the transaction succeeds without partial writes;
- `arkcli-managed-agent` is retired;
- the current `arkcli-agent` is installed and becomes the only managed agent
  Skill name;
- unrelated user-managed `ark-` and `arkcli-` entries remain unchanged.

Repeat the same assertions for BytePlus candidate, integration, and stable
builds. The install contract must not vary by package channel,
embedded/CDN/GitHub source, old manifest product, digest, or local edits.
