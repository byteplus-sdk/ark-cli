---
name: arkcli-agent
version: 0.3.1
description: "Create, inspect, update, debug, and interact with BytePlus Ark Managed Agents and their sessions, skills, files, and MCP integrations."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli agent --help"
---

> **BytePlus support note:** Managed Agent commands are available. `custom`, `ark`, and `all` Skill sources are supported; public Market/SkillHub is not.

# arkcli-agent

**CRITICAL — Before starting, MUST read the documentation for [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md).**

Use this skill to create, query, debug, or interact with ARK Managed Agents. Core principle: use stable product commands and avoid direct use of OpenTOP Actions unless necessary; Session runtime resources, events, threads, and Files APIs should use the data plane directly; MCP OAuth logins should first check the backend provider and then use `arkcli-agent +mcp-login`.

Before execution, read the corresponding reference based on the user's intention; only read the relevant reference and do not load all details at once. If the user's request is clear about creating, copying, attaching files, chatting, or MCP login, complete the necessary confirmation or disambiguation before execution, and do not just provide command suggestions.

## Minimum Decisions for Agent Creation

| Input | Required Handling |
| --- | --- |
| No primary Agent model specified | Start with `arkcli agent model list --format json` and read all candidates; do not add query by default, truncate results, or invent IDs. Fetch exact-version metadata only for explicit context/modality/capability requirements |
| Choose a tool's bound model | Use `agent model list --usage tool --format json` without defaulting to `--primary-only`; place the selected `items[].model` in `Tools[].Configs[].Models[].ID`, not the Agent's primary model |
| No skill specified | Search the current account's custom skills first; `ark` and `all` are available for read-only discovery, while Market/SkillHub is not |
| Local skill zip provided | Run `agent skill create --zip`, then use the returned `skill-...` ID and version when creating the Agent |
| No tools specified | Let the CLI inject the complete default toolset; an explicit `--tool` replaces the full default array |
| No explicit request for multimodal tools | Do not query tool models or add a multimodal toolset, image_generate/video_generate, or their Models. Retain ordinary default tools; a model's multimodal input support is not permission to add generation tools |
| No environment specified | When creating a Session, select the latest environment in the current project; ask for one only if none is available |
| Agent created | Read it back with `agent agent get <agent-id> --format json` and report the final Model, System, Tools, Skills, MCP servers, and extension fields |
| The user expects a reply | Use `+new session ... --message` or `events send ... --stream` for short work; use `--poll` or cursor-based event polling for large or long-running work |

## Select Path

| User Intent | Preferred Command | Details |
| --- | --- | --- |
| Create/Update/Delete Managed Agent | `arkcli agent agent ...` | [`references/agent.md`](references/agent.md) |
| Copy an Existing Agent and Rename/Modify Configuration | `arkcli +new-agent --fork <agent-id> [--name <new-name>]` | [`references/agent.md`](references/agent.md#Copy-agent) |
| Choose Available Models for Creating an Agent | `arkcli agent model list` | [`references/agent.md`](references/agent.md#model selection) |
| Choose Models for Tools Supporting Bindings, Such as Image/Video Generation | `arkcli agent model list --usage tool` | [`references/agent.md`](references/agent.md#discover-tool-models-and-cache-behavior) |
| Query Supported Model Runtime Values and Defaults | `arkcli agent model config <foundation-model-name>` | [`references/model-config.md`](references/model-config.md) |
| Choose a Skill for Creating an Agent | Search custom skills first; BytePlus also supports `--source ark/all`, but not Market/SkillHub | [`references/skills.md`](references/skills.md) |
| Query/Use Custom Skills in the Current Account | First `agent skill list --source custom --limit 100`, continue with `NextPage` if no matches; use `--page-all` or `--skill <skill-id>` for complete candidates | [`references/skills.md`](references/skills.md) |
| Query Ark Skills or All TOP Skills | `agent skill list --source ark` or `--source all`; Ark skills are read-only | [`references/skills.md`](references/skills.md) |
| Upload a Local Custom Skill Zip | `arkcli agent skill create --zip <file>` or `arkcli agent agent create --skill-zip <file>` | [`references/skills.md`](references/skills.md) |
| Scan or Import Skills from GitHub | `agent skill github scan <repo>` / `agent skill github import <repo> --path ...|--all` | [`references/skills.md`](references/skills.md) |
| Manage Custom Skill Versions or Delete a Skill | List all versions first, then delete in dependency order: non-latest versions, latest version, and finally the Skill | [`references/skills.md`](references/skills.md) |
| Create a Session Environment or Session | `arkcli agent env ...` or `arkcli agent session ...` | [`references/session-files.md`](references/session-files.md) |
| Self-hosted environments, queue diagnostics, or stopping work | `agent env create --runtime-type self_hosted` / `agent env work list/get/stats/stop` | [`references/self-hosted.md`](references/self-hosted.md) |
| Set an Environment Initialization Script | `arkcli agent env create/update --setup-script @./bootstrap.sh` | [`references/session-files.md`](references/session-files.md#environment-setup-script) |
| Override an Agent or Environment for One Session | `arkcli agent session create --agent-overrides ... --environment-overrides ...` | [`references/session-files.md`](references/session-files.md#session-overrides) |
| Upgrade an Existing Session's Model, Agent Version, or Runtime Configuration | `arkcli agent session upgrade <session-id>` | [`references/session-upgrade.md`](references/session-upgrade.md) |
| Mount a TOS Directory When Creating a Session | After the user provides the path, use `arkcli agent session create --tos-path tos://<bucket>/<prefix>/`; never guess the bucket or prefix | [`references/session-files.md`](references/session-files.md#session-tos-resource) |
| Continue an Existing Session | `arkcli +new session` | [`references/events-chat.md`](references/events-chat.md) |
| Create a New Session and Chat | `arkcli +new session <agent-id> --environment-id <env-id>` | [`references/events-chat.md`](references/events-chat.md) |
| Send Messages to or Stream Replies from a Session | Plain `events send` is write-only; add `--stream` for SSE replies (`--wait` remains a compatibility alias), use `--poll` for long work, or follow with `+tail`. Streaming requests Agent message/thinking deltas by default; use `--no-event-deltas` for complete events only | [`references/events-chat.md`](references/events-chat.md) |
| Compact Session Context | `arkcli agent session compact <session-id> [--instructions <text|@file>]` | [`references/events-chat.md`](references/events-chat.md) |
| Diagnose or Export a Session | `arkcli +debug <session-id>` or `arkcli +export <session-id>` | [`references/debug-export.md`](references/debug-export.md) |
| Upload a File to an Existing Session | `arkcli agent session resources add <session-id> --path <file>` | [`references/session-files.md`](references/session-files.md) |
| Query or Upload Files via the Files API | `arkcli agent file upload/list/get/wait/delete` | [`references/session-files.md`](references/session-files.md) |
| Manage Memory Stores or Memories | `arkcli agent memory-store ...` | [`references/interfaces-gaps.md`](references/interfaces-gaps.md) |
| Query MCP/Vault/Credential/MCP OAuth Providers | `arkcli agent vault oauth-provider list` or `arkcli agent vault ...` or `arkcli agent +mcp-login ...` | [`references/mcp-vault.md`](references/mcp-vault.md) |

## Model Discovery: Primary Agent vs Tool Models

- Default `--usage agent` selects Agent models using `ep_agent.support`; `--usage tool` uses version-level `epa_tool_model.support` without intersecting the two gates. Neither query defaults to `--primary-only` or excludes candidates by `primary_version`; add that flag only when the user explicitly asks for primary versions.
- Bind the returned `items[].model`, not the internal `id` (such as epm-*). Only configure Models for tools that support bindings; see [binding fields and full Tools replacement](references/agent.md#tool-model-bindings-supported-tools-only).
- Both usages share a 5-minute full-version ArkModels cache and expose `cache.source/fetched_at/expires_at`. Add `--refresh-cache` only when a refresh is needed, not on every lookup. Failures enforce a 1-minute cooldown even for forced refresh; an error is not an empty candidate list.
- Lists do not truncate candidates or expose context/modality/capability filter flags. Only for explicit requirements, query `ListModelMetaDatas` using each candidate's `name` and `version`, then filter. Do not fetch per-model metadata otherwise. See [exact-version filtering](references/agent.md#exact-version-filtering) for fields and missing-data handling.
- `agent model config` queries runtime parameters, not tool eligibility. Explicit model IDs can still be submitted to create/update for backend validation; do not introduce a cache-based write gate. See [discovery parameters and cache boundaries](references/agent.md#discover-tool-models-and-cache-behavior), including query-mode enrichment.

## Authentication and Profiles

- Without an explicit API key, Managed Agent requires a `type=platform` profile; it does not automatically use personal/team plan credentials. A successful login does not establish that the default profile supports Managed Agents.
- When relying on profile credentials, check the type using `auth status --format json`. If the default is a plan, use a verified `--profile <platform-profile>` for this invocation; do not change the global default with `profile use`. If no Platform profile is available or identity/project is ambiguous, follow the Profile Skill and obtain confirmation instead of guessing a name or switching accounts.
- Data-plane commands accept `--api-key` or `ARK_API_KEY` even when the selected profile is a plan. Key priority is `--api-key` > `ARK_API_KEY` > the Platform profile's key. The server validates the supplied key and its permissions; stop on authentication failure without switching accounts.
- Without an explicit URL, MA uses its standard data-plane endpoint for the current product, Region and environment, never the profile's plan or custom route. Key-only data-plane calls also work without a profile. Explicit `--base-url` / `ARK_BASE_URL` wins, but requires an explicit key too; a URL alone cannot replace plan credentials.
- Control-plane calls still require login credentials. Control-plane-only commands do not accept key/URL overrides, and mixed workflows cannot use an API key instead of login. For control-plane work, inspect `arkcli auth status --format json` and handle expired SSO/STS first. Data-plane-only calls with an explicit key do not require login.
- Stop on `managed_agent_profile_required` without using Raw API as a workaround. Offline `--dry-run` does not establish execution eligibility. Overrides affect this invocation only, not the default profile or saved key.
- In online environments, use `--env prod` by default; do not default to `stg`.
- BytePlus `--env stg` selects the control-plane environment; its built-in data-plane URL remains the standard production endpoint. For stg data-plane testing, require a verified stg URL and matching explicit key. An explicit URL is preserved rather than rewritten by environment selection.
- Non-interactive SSO logins are two-step: first `arkcli auth login --no-browser` to get a URL; users need to paste the base64 code to complete the login with `arkcli auth login --no-browser --code <code>`.

## Pagination

- Use the global `--page-all` flag for supported lists. Automatic pagination defaults to 100 items per page and at most 10 pages. `--page-limit <N>` sets the maximum number of pages to fetch, not the number of items per page; use the command's local page-size flag (such as `--limit`, where supported) to change items per page. `--page-delay <ms>` controls the interval between pages.
- Supported: Agent/versions, Env, Session, Skill (`custom`, `ark`, and `all`; BytePlus does not expose Market/SkillHub), Memory Stores/Memories, Vaults/Credentials/OAuth Providers, Files, Session Events/Threads. The CLI uses the backend pagination mechanisms (`Page`, `PageNumber`, `PageToken`, `after`) and merges the results.
- `agent model list`, `memory-store creators`, and `session resources list` do not have pagination mechanisms; do not assume `--page-all` will complete the results. If `--page-limit` is set, check the `NextPage`, `has_more`, or `TotalCount` in the response to determine if more data needs to be fetched.

## Confirmation for Deletion

- The destructive `delete` command for Managed Agents will display an irreversible warning and ask for confirmation (`[y/N]`) if not passed `--yes` in a real TTY; only `y/yes` will trigger the backend, other inputs will cancel.
- Non-interactive environments (AI Agents, CI, pipelines) do not read from stdin; if not passed `--yes`, return `type=requires_confirmation` and do not call the backend. Only after the user confirms deletion can the caller retry with `--yes`.
- `--dry-run` is not domain-wide. Use it only when the leaf command's
  `--help` lists it. Current support includes locally deterministic
  Env/Session/Memory/Vault/Credential writes, `agent agent create/update/delete`,
  `agent skill create/update/delete`, `agent file upload/delete`, and the local
  file writers `agent skill download` and `+export`. `+new-agent`, `+iterate`,
  MCP login, and all pure read commands do not register it.
  Client Preview only produces a zero-network `preview.v1` plan; it does not
  replace entitlement, version/dependency checks, or destructive confirmation
  during real execution.

## Long-Running Workflow Rules

- Execute one `arkcli` command per shell or tool call. Do not combine Session creation, ID extraction, event sending, and event polling with `&&`, `;`, pipes, or heredocs.
- After a successful write, capture the ID from structured output and use that literal ID in the next command.
- If a write times out, first use a separate read command such as `session list/get`, `events list`, or `+debug` to determine whether it succeeded. Do not blindly retry a request whose result is unknown.
- Retry network interruptions, 429 responses, and 5xx responses only, with bounded exponential backoff. Do not retry validation, authentication, entitlement, permission, or explicit business errors.
- `events send --stream` automatically falls back from SSE to cursor-based event-list polling after its stream timeout. `--wait` remains a compatibility alias. If both stages time out, continue from the reported cursor instead of sending the user's message again.

## Command Quick Reference

| Command | Description |
| --- | --- |
| `arkcli agent agent list/get/create/update/delete/versions` | CRUD for Managed Agents and versions |
| `arkcli agent model list` | Default: primary models whose `items[].model` goes into `--model`. With `--usage tool`, use it for `Configs[].Models[].ID`. Optional `--query` enriches details and ranking |
| `arkcli +new-agent` | Enhanced entry for creating an Agent; supports `--fork/--from` to copy an existing Agent and create a new one |
| `arkcli +iterate` | Update Agent configuration, create a new Session, and run one-shot/REPL; `--environment-id/--env-id` can be omitted, and the latest environment will be chosen automatically |
| `arkcli agent skill search/list/get/create/update/delete/versions/download/set-protection/github` | Query custom/Ark skills, import GitHub skills, control custom-skill protection, manage versions, or delete in dependency order; BytePlus defaults `list/search` to `--source custom` and rejects only Market/SkillHub |
| `arkcli agent env list/get/create/update/delete` | CRUD for Environment |
| `arkcli agent session list/get/create/update/delete` | CRUD for Session |
| `arkcli agent session upgrade <session-id>` | Upgrade runtime configuration in place using snake_case; read back to verify completion |
| `arkcli agent session resources list/add/get` | CRUD for data plane session resources; `get` is a local filter based on `list` |
| `arkcli agent session events list/send/stream` | Data plane events; streaming requests Event Deltas by default and `--no-event-deltas` falls back to complete events. `user.custom_tool_result` requires `custom_tool_use_id`; `user.tool_result` is self-hosted only |
| `arkcli agent session compact <session-id>` | Actively compact context; normal thread idle/end_turn plus session idle/end_turn completes the command, while a compacted event is optional confirmation of actual compaction |
| `arkcli agent session threads list/get` | CRUD for data plane threads |
| `arkcli agent file list/get/upload/wait/delete` | CRUD for Files API |
| `arkcli agent memory-store list/get/create/update/delete` | CRUD for Memory Stores |
| `arkcli agent memory-store memories list/get/create/batch-create/update/delete` | CRUD for Memories |
| `arkcli agent vault list/get/create/update/delete` | CRUD for Vaults |
| `arkcli agent vault oauth-provider list` | Query registered MCP Providers; the MCP server information can be used in the Agent `--mcp-server` |
| `arkcli agent vault credentials list/get/create/update/delete` | CRUD for Credentials |
| `arkcli agent +mcp-login` | MCP OAuth login: local callback + CreateVaultOAuthFlow + wait for credential creation |
| `arkcli +chat <prompt>` | Quick dialog for Responses API; do not use it as a Managed Agent session entry |
| `arkcli +tail <session-id>` | Human-readable event stream |
| `arkcli +new session` | Session selector for Managed Agents; can continue an existing session or create a new one |
| `arkcli +new session <agent-id> --environment-id <env-id>` | Direct entry for creating a new session and running one-shot/REPL; always creates a new session first |
| `arkcli +debug <session-id>` | Diagnose the session, events, resources, and threads |
| `arkcli +export <session-id>` | Export the session as a tar.gz |
