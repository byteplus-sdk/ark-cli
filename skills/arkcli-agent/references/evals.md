# arkcli-agent 2026-08 evaluations

## Model runtime metadata

- "What thinking modes and service tiers does this model support?": resolve the selected item's `name`, call `agent model config <name>`, explain the parameters, and report supported/default values from `Key/Data`.
- "Query thinking only": support repeated and comma-separated `--key`; do not invent a version flag or substitute model-list results for metadata.
- "Keep my settings after changing models": query the new model, explain incompatibilities, and neither inject defaults nor add client-side business validation.
- "Empty metadata or an unknown key": report it without inventing enums; preserve future keys and values without an allowlist.
- Use a verified Platform profile in the current BytePlus product without cross-product fallback.

## Profile type boundary

- The default is a personal/team Coding Plan and the user wants MA: use an explicitly verified Platform profile for this invocation; do not silently change the default, invent profile names, or switch accounts.
- Only an explicit key is supplied with a plan profile or no profile: use the standard BytePlus MA endpoint, without requiring a URL, inheriting a plan route, or changing saved profiles.
- An explicit key and URL are supplied with a plan profile: preserve the URL and allow data-plane execution; control-plane steps still require login. Reject a URL without an explicit key. Report server authentication failures without credential fallback. For BytePlus stg, require a verified data-plane URL instead of assuming `--env stg` changes the standard data-plane endpoint.
- Preview succeeds under a plan profile: report an offline plan only, not execution eligibility. Stop on `managed_agent_profile_required` without replaying writes.

## Existing Session upgrades

- "Preview changing this existing Session's model": use session upgrade --model ... --dry-run with zero network; do not update the Agent resource, recreate the Session, or inject compact.
- "Use the latest Agent version with my chosen overrides": resolve an actual published version, supply --agent-version, explain baseline replacement, and confirm before execution. Omission is not latest.
- "Clear Skills but keep tools": --agent '{skills: []}', without default tools; null is not clear. File/inline paths preserve snake_case and arbitrary tool schema property names.
- "Update environment variables/packages": explain merge semantics and that empty objects do not clear old entries; do not send Sandbox fields for self-hosted or switch environment IDs.
- "Refresh Vault": send only the existing binding set; do not create credentials or change bindings. Initial events are optional and --compact is opt-in.
- "It returned upgrading, did it succeed?": report acceptance only, read back status and target model/version/configuration, and do not invent task polling. Inspect ambiguous outcomes before retrying and do not send messages during upgrade.
- "Use model connection settings": upgrade supports provider/protocol/base_url/headers; do not apply creation-override filtering. Keep Authorization/API Keys out of preview output and reports.

## Self-hosted operations

- Create a self-hosted environment: use `env create --runtime-type self_hosted`, preview at the leaf, confirm, create, read back, and explicitly bind a Session; do not inject cloud networking/packages.
- Diagnose stalled work: read work stats/list/get without dry-run, check next_page, and do not equate pending with running or infer business success/failure from workers_polling=0 or stopped alone.
- Preview stopping work: run only work stop --dry-run and inspect preview.v1; do not add --yes or send a stop request.
- Stop work: confirm target and impact before work stop --yes; do not automatically force. Read back state without claiming the Sandbox has exited. Do not invent stop reason or Worker Credential management flags.

Use these cases to verify Agent model fields, Skill source boundaries, GitHub import, protection, and active compaction.

## Agent configuration

- “Create an Agent with a get_weather custom tool”: use `--tool` with `Type: custom`; explain Name, Description, InputSchema.Type, Properties, and Required. InputSchema is an object; nested schema keys and parameter names remain unchanged. Do not confuse tools with Skills or invent `--custom-tools`.
- “Add a custom tool and keep existing tools”: read current configuration and supply the full Tools array; explain replacement, repeated flags, and clearing with `[]`. A declaration does not execute a function; return results with the matching `custom_tool_use_id`. Business limits remain server-validated.
- "Change the Agent description; I do not know its version": omit `--version`; real execution reads `GetAgent` once and uses the current version for `UpdateAgent`, without asking the user to look it up.
- "Update against the version I read": preserve an explicit positive `--version` or file/stdin `Version`, with flag precedence, even when model/Metadata hydration requires a lookup.
- Without a version, dry-run remains offline and shows two steps with `<current-agent-version>` unresolved. Failed reads or invalid current versions prevent writes; version conflicts never trigger automatic version refresh and write retry. Session version semantics do not change.

Request:

> Create an Agent displayed as “Reasoning Assistant”, enable thinking, use high reasoning effort, and do not specify speed.

Expected behavior:

- Use `--display-name`, `--thinking`, and `--reasoning-effort`; do not inject `speed=standard`.
- Inspect `agent agent create ... --dry-run --format json` before the real write.
- A runtime-only model update does not require the model ID again: real execution calls `GetAgent` once and puts the current `Model.Id` into `UpdateAgent`, while dry-run stays offline and shows the two-step plan with an unresolved placeholder.
- If the user explicitly supplies `--model model-new` or a new `Model.Id` through an object/file, update to that model without looking up and overwriting it with the old ID. A dedicated flag still overrides the same field inside `--model`.
- Do not call `UpdateAgent` if the current Agent omits `Model.Id`; return a clear error instead.
- Clear the display name with `agent agent update <id> --display-name ""`.

## Session model overrides

Request:

> Create a Session from an existing Agent, but select another Managed Agent model and override thinking, reasoning effort, and speed.

Expected behavior:

- Put the selected model in `Model.Id` inside `--agent-overrides`; do not reject it merely because it differs from the base Agent model.
- Choose `Thinking`, `ReasoningEffort`, and `ServiceTier` from the selected model's metadata, not from assumptions about the base model.
- Use only `auto`, `default`, or `fast` for `ServiceTier`; never generate `priority`, and omit `fast` when the selected model does not support it.
- Read back `Result.Agent.Model` after creation to verify the model ID and runtime parameters. Resolving `auto` to the effective `default` does not mean the field was dropped. Send a minimal message to verify inference with the selected model.
- Keep only `Id/Speed/Thinking/ReasoningEffort/ServiceTier` in the final request's `Model`; filter `Provider/Protocol/BaseUrl/Headers` and any other undefined fields without a local error.
- Inline overrides, `@file`, and `AgentWithOverrides` in the general `session create --file` payload produce the same canonical request. Explicit empty overrides return a validation error and never panic.
- A runtime-model-only `+iterate` update hydrates the current `Model.Id`. With a Session Agent override, keep the version in `AgentWithOverrides.Version` and omit top-level `AgentId/AgentVersion`. A display-name-only change must still execute UpdateAgent.

## Skill sources and versions

Request:

> List custom and Ark Skills, then attach a bare custom Skill ID to an Agent.

Expected behavior:

- Use `agent skill list --source all`; the TOP request omits `Source`.
- Omit `Version` for a bare `skill-...`; never send the literal `latest`.
- For a specific custom Skill download, use the exact `Version` from `versions`, never an invented semantic version or `VersionId`. Omit `--version` for latest; never use an unresolved preview placeholder in a real download.
- Ark Skills support get/list/versions but reject update/delete/download/set-protection.
- BytePlus supports `custom`, `ark`, and `all`; only Market/SkillHub is rejected.

## GitHub import and protection

Request:

> Import every Skill from `npx skills add owner/repo` and enable protection.

Expected behavior:

- Pass the complete `npx skills add owner/repo` input unchanged to `ScanGithubSkill.Input`; do not validate it locally and never execute npx.
- Scan first, then import valid candidates with `--all --protection-enabled=true`.
- `--path` conflicts with `--all`; `--selected` conflicts with global protection.
- Print every item status and exit nonzero after output if any item is `failed`.
- `--dry-run` is zero-network and keeps scan-dependent values unresolved.

## Tool model bindings

Default creation scenario: the user requests an ordinary Agent or selects an image-input-capable model without requesting multimodal tools. Do not discover tool models or add a multimodal toolset or `Configs[].Models`; retain ordinary default tools. Contrast: enter tool-model configuration only for explicitly requested image/video generation or user-supplied configurations, without adding another unrequested generation tool.

Request:

> Configure two models for a multimodal Agent's image_generate tool, retain other tools, then copy the Agent.

Expected behavior:

- Get the existing configuration; set exact IDs in `Configs[].Models` under `agent_toolset_multimodal_20260825`, with exactly one `Default: true`. Do not change the primary model.
- Explain that only this toolset's `image_generate` and `video_generate` support bindings under the current contract. Never inject Models into ordinary/custom/MCP tools or DefaultConfig.
- Submit the full Tools replacement, retaining other configurations. Preview preserves explicit false and display Name/Version without claiming server validation.
- Obtain confirmation before the real update, read back, and use `+new-agent --fork` for copying without losing Models. Report backend rejection without silently removing bindings and retrying.

## Tool model discovery and caching

Additional scenario: the user requests Agent or tool models without a version restriction, and eligible candidates include both primary and non-primary versions. Neither usage adds `--primary-only`, drops non-primary versions locally, or defaults selection to the primary version. Add the flag only when the user explicitly requests primary versions.

Request:

> Discover tool-eligible models, including eligible non-primary versions, after a recent primary-Agent model lookup.

Expected behavior:

- Use `agent model list --usage tool` without defaulting to `--primary-only` or intersecting with `ep_agent.support`. Bind `items[].model`, not internal epm-* IDs.
- Share the full-version ArkModels cache across usages; name/usage/primary filtering stays local without poisoning other queries. Reuse fresh data for 5 minutes and coalesce cross-process refreshes.
- Use `--include-all` only for diagnosis, not to select entries with `tool_model_support=false`; refresh explicitly with `--refresh-cache`.
- Report rate limiting/failure instead of claiming no candidates; respect the 1-minute cooldown even for forced refresh. Do not present expired eligibility as current or substitute `agent model config`.
- Do not automatically activate/bind models or add a cache-based create/update gate. The backend still validates writes with explicitly supplied IDs.

## Full model lists and optional metadata filtering

- For a plain list, return all eligible versions without size or hard-filter flags, default query/primary-only, or per-model metadata calls.
- For an explicit minimum of 200000 context tokens and image input, first list, then query `ListModelMetaDatas` for each candidate's name + version with `context_window` and `input_modalities`. Do not substitute `agent model config`.
- Same-name v1 has 128000 tokens and v2 has 256000: only v2 meets the threshold. Do not copy primary-version query detail to another version.
- Missing/failed metadata is unknown, not zero or verified compliance. Disclose unknown entries and recommend only verified matches for strict requirements; distinguish lookup failure from no matches.
- Optional query enriches/ranks without dropping unmatched candidates. Generic `models search` retains its own filtering flags.

## Active compaction

Request:

> Compact a Session with the instruction `Keep <key findings> & TODOs`.

Expected behavior:

- Use `agent session compact <id> --instructions ...` and preflight an idle Session.
- XML-escape instructions in the slash envelope.
- Treat normal thread idle/end_turn plus session idle/end_turn as success. A compacted event confirms actual compaction when present; its absence must not fail the command or be reported as confirmed history compaction.
- Return nonzero for failure/termination or timeout, and never treat send acceptance alone as completion.
- REPL `/compact <instructions>` retains its arguments; `/clear` remains unchanged.
