# Skills

## Skill Selection

- BytePlus exposes TOP `custom`, `ark`, and `all` sources. `agent skill search/list` defaults to `--source custom`; `ark` queries Ark-provided skills and `all` omits `Source`. Only `market`, `skillhub`, and `skill_hub` are rejected before any remote request.
- Market/SkillHub skill is **not available in BytePlus**. Ark skills can be discovered and attached, but they are read-only: update, delete, download, and protection changes are rejected by the CLI.
- Custom skill: The default selection process for skills within the user's account involves pagination: fetch the first page of 100 items, and the AI agent within the CLI calls `ListSkillsForTop` to filter by name, description, and capability. If no suitable candidate is found, the `NextPage` is used to fetch the next page until a match is found or no further pages are available. Do not filter by `Name` on the server side or pull the entire catalog in one go. The internal CLI entry point is `ListSkillsForTop`, and the actual top-level action is `ListSkills`. Only use the `skill-...` IDs returned by this interface; `s-...` IDs are not used for custom skills.
- When creating an Agent, a bare custom ID can be passed directly as `--skill skill-xxx`. The CLI omits `Version`, allowing the service to use the latest version; it never sends the literal string `latest`. A bare `--skill s-xxx` is rejected in BytePlus.
- When uploading a custom skill, it goes through the data plane `POST /api/v3/skills`, not the top-level `CreateSkill`. A current profile with an available ARK API key is required.
- `agent skill get <id>` and `versions` read custom or Ark skills through TOP. List summaries preserve `DisplayTitle`, `Source`, `UpdateTime`, `Name`, and `ProtectionEnabled`; Agent Skill responses preserve `UseLatest`, concrete `Version`, and `DisplayName`.
- `agent skill create --protection-enabled` sets protection during upload. Use `agent skill set-protection <id> --enabled=true|false` for an existing custom skill; explicit `false` is preserved and sent through TOP `UpdateSkill`.
- `agent skill update <id> --zip <file>` creates a new version through OpenTOP `CreateSkillVersion`; it does not modify an existing version in place. Use `agent skill versions <id>` to list versions.
- `agent skill download <id> [--version <version>]` resolves the requested or latest version, obtains `PreSignedTOSURL` through `GetSkillVersion`, and downloads it with an ordinary unauthenticated GET. The default filename is `<skill-id>-v<version>.zip`.
- `agent skill download ... --dry-run` previews version resolution, download, and local save steps without network or filesystem writes. If `--version` or `--output` is omitted, treat the corresponding `unresolved` value as a placeholder; never claim it was resolved online and never create or replace a file during Preview.
- `agent skill delete <id>` calls `DeleteSkill`. It does not cascade to SkillVersions and may run only after every version has been deleted.
- `agent skill delete <id> --version <version>` calls `DeleteSkillVersion` and deletes only that version. Before deleting the latest version, delete every non-latest version.
- `agent skill create/update/delete` support command-local `--dry-run` for a zero-network preview of the upload or delete request. Delete preview does not require `--yes`, but it does not validate the online version dependency order. Real deletion still requires a complete version-list read and interactive confirmation or explicit `--yes`.
- BytePlus does not expose a market skill selection flow. Custom/Ark selection uses `Items[].Id`, `Name`, `DisplayTitle`, `Description`, `Source`, and version fields from TOP `ListSkills`.
- When selecting a custom skill, read the first page of complete `Items`, and compare `Id`, `Name`, `Description`, and `LatestVersion`. Do not rely solely on the service-end keywords or the first entry. If there are not enough relevant candidates, use `NextPage` to fetch the next page. The `AgentSkills` only contain `skill-...` custom skills. After confirming the candidates, they can be fetched directly or pass the bare `--skill <Items[].Id>`.
- When multiple candidates are close, list them for the user to choose from. When the user requests automatic completion, select the skill with the highest relevance and the most explicit version.

- Pagination selection prioritizes "on-demand fetching": the first page uses `--limit 100`, and the next page's `NextPage` is passed directly to `--page`. Only when the user requests a complete list or needs offline analysis of all skills, or when no pages are hit, use `--page-all`. It is subject to the global `--page-limit` and cannot treat truncated catalogs as complete candidates.

- Do not mix `Items[].Id` and `LatestVersionStatus.VersionId`: for creating an Agent, pass `SkillId` and the semantic version `Version`, not `VersionId`.

## Import from GitHub

GitHub import uses only the public repository's default-branch HEAD. There are no private-repository, branch, or tag options:

```bash
arkcli agent skill github scan owner/repo --format json
arkcli agent skill github import https://github.com/owner/repo --path skills/foo --path skills/bar --format json
arkcli agent skill github import 'npx skills add owner/repo' --all --protection-enabled=true --format json
arkcli agent skill github import owner/repo --selected @./selected.json --format json
```

- When constructing a command, prefer the input forms known to the service: `owner/repo`, a complete GitHub HTTPS URL, or a complete `npx skills add ...` string. These are invocation guidelines, not an Agent-side or CLI-side allowlist.
- Once the user supplies repository input, do not parse, rewrite, or reject it in the Agent or CLI. Pass it unchanged to `ScanGithubSkill.Input` and let the service interpret and validate it. The CLI never starts a shell or executes npx from the input.
- Repeat `--path`, or use `--all`; they conflict and selections must be valid scan candidates.
- `--selected` reads an array of `{Path, ProtectionEnabled}` objects and conflicts with global `--protection-enabled`.
- scan is read-only. import uses Workflow Client Preview; `--dry-run` sends no network requests and leaves scan-dependent values unresolved.
- All item results (`ok`, `updated`, `skipped_duplicate`, `failed`) are printed. Any failed item makes the command exit nonzero after preserving the complete result.

## Custom Skill Deletion Order

Skill deletion has a strict dependency order. Before deleting anything, use `agent skill versions <id>` to fetch the complete version list and identify the latest version:

1. To delete the entire Skill, delete every non-latest SkillVersion first, delete the latest SkillVersion next, confirm that the version list is empty, and then delete the Skill.
2. To delete the latest SkillVersion, delete every non-latest SkillVersion first. Delete the latest version only after confirming that it is the only remaining version.
3. To delete one non-latest SkillVersion, delete that version directly and then fetch the version list again to verify the result.

Re-run `agent skill versions <id>` after every deletion. Do not assume that `DeleteSkill` cascades to versions, and do not call it while any SkillVersion remains. Run each destructive command separately and follow the interactive confirmation or `--yes` requirement.

## Common Commands

```bash
arkcli agent skill search "Excel data analysis" --source custom --limit 10 --format json
arkcli agent skill list --source custom --query "Excel data analysis" --limit 20 --format json
arkcli agent skill list --source custom --name "My analysis data" --limit 20 --format json
arkcli agent skill list --source custom --limit 100 --format json
# If the previous page returns `NextPage` and no suitable candidate is found, continue:
arkcli agent skill list --source custom --limit 100 --page '<NextPage>' --format json
# Note: --source market is not supported on BytePlus; custom (default), ark, and all are supported.
arkcli agent skill create --zip ./my-skill.zip --display-title "My Skill" --format json
arkcli agent skill create --zip ./my-skill.zip --display-title "My Skill" --protection-enabled=true --format json
arkcli agent skill set-protection skill-xxx --enabled=false --format json
arkcli agent skill list --source ark --limit 100 --format json
arkcli agent skill list --source all --limit 100 --format json
arkcli agent skill github scan owner/repo --format json
arkcli agent skill github import owner/repo --all --format json
arkcli agent agent create --name arkcli-local-skill-agent --model <items[].model from agent model list> --skill-zip ./my-skill.zip --format json
arkcli agent agent create --name arkcli-existing-custom-skill-agent --model <items[].model from agent model list> --skill skill-xxx --format json
arkcli agent skill update skill-xxx --zip ./my-skill-v2.zip --format json
arkcli --page-all agent skill versions skill-xxx --format json
# Replace the placeholder with the selected entry's exact Version from the versions response:
arkcli agent skill download skill-xxx --version '<Version-from-versions>' --output ./skill-selected-version.zip --format json
# Omit --version when the user wants latest; the CLI resolves LatestVersion:
arkcli agent skill download skill-xxx --output ./skill-latest.zip --format json
# To delete a Skill, delete each non-latest version first:
arkcli agent skill delete skill-xxx --version <non-latest-version> --yes --format json
# Fetch versions again and delete latest only after it is the sole remaining version:
arkcli --page-all agent skill versions skill-xxx --format json
arkcli agent skill delete skill-xxx --version <latest-version> --yes --format json
# Fetch versions again and delete the Skill only after the list is empty:
arkcli --page-all agent skill versions skill-xxx --format json
arkcli agent skill delete skill-xxx --yes --format json
```
