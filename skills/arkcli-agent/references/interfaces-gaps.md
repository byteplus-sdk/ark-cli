# Interface Links and Gap

## Interface Links

- Agent: `CreateAgent` / `GetAgent` / `ListAgents` / `UpdateAgent` / `DeleteAgent` / `ListAgentVersions`
- Skill: data-plane `POST /api/v3/skills` for `CreateSkill` + TOP `Get/List/DeleteSkill` and `Create/List/Get/DeleteSkillVersion` (BytePlus supports custom skills only; SkillHub / market skill via `ListMarketSkills` is unavailable)
- Env: `CreateEnvironment` / `GetEnvironment` / `ListEnvironments` / `UpdateEnvironment` / `DeleteEnvironment`
- Session: `CreateSession` / `GetSession` / `ListSessions` / `UpdateSession` / `DeleteSession`
- Session data-plane: `GET/POST /api/v3/sessions/:session_id/resources`, `GET/POST /api/v3/sessions/:session_id/events`, `GET /api/v3/sessions/:session_id/events/stream`, `GET /api/v3/sessions/:session_id/threads`, `GET /api/v3/sessions/:session_id/threads/:thread_id`
- Files data-plane: `GET/POST /api/v3/files`, `GET/DELETE /api/v3/files/:file_id`
- Memory: `ListMemoryStores` / `CreateMemoryStore` / `ListMemories` / `CreateMemory` etc.
- Vault: `ListVaults` / `CreateVault` / `ListOAuthProviders` / `CreateVaultOAuthFlow` / `ListCredentials` / `CreateCredential` etc.

## Aligned / Acceptable Alternatives

- Agent / Env / Session / Memory / Vault / Credential CRUD commands are already available.
- Environment setup scripts are supported through `--setup-script`, which writes `Config.SetupScript` and accepts `@file`.
- Session creation, `+new session`, and `+iterate` support `AgentWithOverrides` and Environment overrides through `--agent-overrides` and `--environment-overrides`.
- Session creation, `+new session`, and `+iterate` support mounting an existing TOS directory through `--tos-path`.
- Environment-variable credentials and managed OAuth refresh fields are supported by typed flags. Sensitive values accept `@file`.
- Skill searches use TOP `ListSkillsForTop` on custom skills only (SkillHub / market skill unavailable in BytePlus); custom skill zips are uploaded via the data plane `POST /api/v3/skills`, and custom skill version updates, listing, downloads, and deletions use OpenTOP Skill/SkillVersion actions.
- Files API already has `list/get/upload/wait/delete` commands; `session resources add --path` can automatically upload -> wait active -> mount.
- Session events/list/stream, threads/list/get are already implemented directly on the data plane, without relying on ArkBFF.
- `ListSessionsForTop` is aligned for session lists: `--agent-id` sends `AgentIds`, `--page/--limit` sends `PageNumber/PageSize`, and `--page-all` fetches continuous pages.
- `+tail` already provides human-readable pretty output; `+new session` without parameters is the PRD session selector entry, allowing continuation of existing sessions or creation of a new session with an agent or environment; `+new session <agent-id> --environment-id <env-id>` creates a one-shot/stdin/TTY REPL session. `+new session`, `+iterate`, and `+tail` recover stream interruptions through event-list compensation.
- `+debug` already aggregates session/events/resources/threads; `+export` already exports a visible diagnostic package.
- `+new-agent --fork` already supports copying existing Agents, naming them with `copy-<source-name>` by default.
- `+iterate` already supports getting the current Agent version, optionally updating it, creating a session, and using `--message` for one-shot/TTY REPL; `--environment-id/--env-id` is optional, and if not provided, the most recently created environment is chosen automatically.
- Default Tools are aligned with the current implementation: no `--tool` sends `agent_toolset_20260701`, containing bash/read/write/edit/glob/grep/web_fetch/web_search; a `--tool` sends the full array to fully override.

## Current Gaps

- The PRD `arkcli +new-agent "..."` CLI internal LLM draft + confirm is not implemented; this is currently handled by an AI agent that understands natural language, selects models/skills/tools, and then calls the structured `+new-agent` or `agent agent create`.
- The PRD `+outcome` shortcut is not implemented.
- The session selector for `+new session` has been superseded by the P0 interactive link, but token counts and session counts in the last 7 days are dependent on backend returns; only the interface-retrievable `id/name/title/status/time/version` fields are displayed.
- The `+iterate` feature is not yet implemented for the PRD TTY environment/resource selector and rich diff; when omitted, the most recently created environment is chosen automatically, and `--diff` outputs a structured preview.
- Session resources native `get/update/delete` are not fully exposed; the CLI `get` is derived from a list, and updates/deletes are unsupported.
- `session resources add` only supports file/local path experiences; `github_repository`, complex `memory_store`, etc., resources must be hand-written in the payload or `session create --resource`.
- Event sending supports ordered `--events` arrays, typed tool results/confirmations, and `--image`/`--document` upload helpers.
- `+export` does not yet provide an available read interface for workspace tarball / memory snapshot; only the manifest is marked as unsupported.
- `arkcli agent +mcp-login` only works for the backend provider list with `CredentialType=mcp_oauth`; static bearer providers use manual credential creation. Notion / Lark Base providers are still limited by the backend metadata discovery availability.
- The inline/local skill directory structure in the PRD examples is not directly supported by `--skill '{type:inline,...}'`; the recommended path is to upload a local zip with `agent skill create --zip` or `agent agent create --skill-zip`.
- `agent agent list` defaults to fetching a single page; for fetching all candidates, `--page-all` is needed, and `--page-limit` is set based on data volume. If multiple candidates are returned, users need to confirm.
- Data plane streaming supports repeated `event_deltas=agent.message` and `event_deltas=agent.thinking` parameters. `events stream`, `+tail`, `events send --wait`, `+new session`, and `+iterate` enable them by default; `--no-event-deltas` requests complete events only. Event-list compensation remains cursor-based and does not request deltas.
