# Skills

## Skill Selection

- BytePlus only exposes `custom` skills. `agent skill search/list` defaults to `--source custom`; passing `--source market`, `skillhub`, or `skill_hub` is rejected before any remote request.
- Market/SkillHub skill: **not available in BytePlus**. If custom does not match, upload a local skill zip via `arkcli agent skill create --zip` or explain no matching custom skill.
- Custom skill: The default selection process for skills within the user's account involves pagination: fetch the first page of 100 items, and the AI agent within the CLI calls `ListSkillsForTop` to filter by name, description, and capability. If no suitable candidate is found, the `NextPage` is used to fetch the next page until a match is found or no further pages are available. Do not filter by `Name` on the server side or pull the entire catalog in one go. The internal CLI entry point is `ListSkillsForTop`, and the actual top-level action is `ListSkills`. Only use the `skill-...` IDs returned by this interface; `s-...` IDs are not used for custom skills.
- When creating an Agent, the bare `custom` ID can be passed directly: `--skill skill-xxx`, and the CLI will fetch `LatestVersion` by calling `ListSkillsForTop`. A bare `--skill s-xxx` is rejected in BytePlus (SkillHub / `s-...` IDs are not available).
- When uploading a custom skill, it goes through the data plane `POST /api/v3/skills`, not the top-level `CreateSkill`. A current profile with an available ARK API key is required.
- `agent skill get <id>` calls the top-level `GetSkill` to fetch the custom skill within the user's account. To list all custom skills within the user's account, use `list --source custom`. Do not use this command to fetch market skill details.
- `agent skill update <id> --zip <file>` creates a new version through OpenTOP `CreateSkillVersion`; it does not modify an existing version in place. Use `agent skill versions <id>` to list versions.
- `agent skill download <id> [--version <version>]` resolves the requested or latest version, obtains `PreSignedTOSURL` through `GetSkillVersion`, and downloads it with an ordinary unauthenticated GET. The default filename is `<skill-id>-v<version>.zip`.
- `agent skill download ... --dry-run` previews version resolution, download, and local save steps without network or filesystem writes. If `--version` or `--output` is omitted, treat the corresponding `unresolved` value as a placeholder; never claim it was resolved online and never create or replace a file during Preview.
- `agent skill delete <id>` calls `DeleteSkill`. It does not cascade to SkillVersions and may run only after every version has been deleted.
- `agent skill delete <id> --version <version>` calls `DeleteSkillVersion` and deletes only that version. Before deleting the latest version, delete every non-latest version.
- `agent skill create/update/delete` support command-local `--dry-run` for a zero-network preview of the upload or delete request. Delete preview does not require `--yes`, but it does not validate the online version dependency order. Real deletion still requires a complete version-list read and interactive confirmation or explicit `--yes`.
- BytePlus does not expose a market skill selection flow. Custom skill selection uses `Items[].Id`, `Name`, `Description`, and `LatestVersion` from TOP `ListSkills`.
- When selecting a custom skill, read the first page of complete `Items`, and compare `Id`, `Name`, `Description`, and `LatestVersion`. Do not rely solely on the service-end keywords or the first entry. If there are not enough relevant candidates, use `NextPage` to fetch the next page. The `AgentSkills` only contain `skill-...` custom skills. After confirming the candidates, they can be fetched directly or pass the bare `--skill <Items[].Id>`.
- When multiple candidates are close, list them for the user to choose from. When the user requests automatic completion, select the skill with the highest relevance and the most explicit version.

- Pagination selection prioritizes "on-demand fetching": the first page uses `--limit 100`, and the next page's `NextPage` is passed directly to `--page`. Only when the user requests a complete list or needs offline analysis of all skills, or when no pages are hit, use `--page-all`. It is subject to the global `--page-limit` and cannot treat truncated catalogs as complete candidates.

- Do not mix `Items[].Id` and `LatestVersionStatus.VersionId`: for creating an Agent, pass `SkillId` and the semantic version `Version`, not `VersionId`.

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
# Note: --source market is not supported on BytePlus; use --source custom (default) only.
arkcli agent skill create --zip ./my-skill.zip --display-title "My Skill" --format json
arkcli agent agent create --name arkcli-local-skill-agent --model <items[].model from agent model list> --skill-zip ./my-skill.zip --format json
arkcli agent agent create --name arkcli-existing-custom-skill-agent --model <items[].model from agent model list> --skill skill-xxx --format json
arkcli agent skill update skill-xxx --zip ./my-skill-v2.zip --format json
arkcli --page-all agent skill versions skill-xxx --format json
arkcli agent skill download skill-xxx --version 2.0.0 --output ./skill-xxx-v2.zip --format json
# To delete a Skill, delete each non-latest version first:
arkcli agent skill delete skill-xxx --version <non-latest-version> --yes --format json
# Fetch versions again and delete latest only after it is the sole remaining version:
arkcli --page-all agent skill versions skill-xxx --format json
arkcli agent skill delete skill-xxx --version <latest-version> --yes --format json
# Fetch versions again and delete the Skill only after the list is empty:
arkcli --page-all agent skill versions skill-xxx --format json
arkcli agent skill delete skill-xxx --yes --format json
```
