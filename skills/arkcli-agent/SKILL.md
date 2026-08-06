---
name: arkcli-agent
version: 0.3.1
description: "Create, inspect, update, debug, and interact with BytePlus Ark Managed Agents and their sessions, skills, files, and MCP integrations."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli agent --help"
---

> **BytePlus support note:** Managed Agent commands are available. Agent skill selection is custom-only; Market/SkillHub sources are not supported.

# arkcli-agent

**CRITICAL — Before starting, MUST read the documentation for [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md).**

Use this skill to create, query, debug, or interact with ARK Managed Agents. Core principle: use stable product commands and avoid direct use of OpenTOP Actions unless necessary; Session runtime resources, events, threads, and Files APIs should use the data plane directly; MCP OAuth logins should first check the backend provider and then use `arkcli-agent +mcp-login`.

Before execution, read the corresponding reference based on the user's intention; only read the relevant reference and do not load all details at once. If the user's request is clear about creating, copying, attaching files, chatting, or MCP login, complete the necessary confirmation or disambiguation before execution, and do not just provide command suggestions.

## Minimum Decisions for Agent Creation

| Input | Required Handling |
| --- | --- |
| No model specified | Use `agent model list --query` and select an exact `items[].model` from the Managed Agent allowlist; never invent a model ID |
| No skill specified | Search the current account's custom skills only; BytePlus does not provide Market/SkillHub skills |
| Local skill zip provided | Run `agent skill create --zip`, then use the returned `skill-...` ID and version when creating the Agent |
| No tools specified | Let the CLI inject the complete default toolset; an explicit `--tool` replaces the full default array |
| No environment specified | When creating a Session, select the latest environment in the current project; ask for one only if none is available |
| Agent created | Read it back with `agent agent get <agent-id> --format json` and report the final Model, System, Tools, Skills, MCP servers, and extension fields |
| The user expects a reply | Use `+new session ... --message` or `events send ... --stream` for short work; use `--poll` or cursor-based event polling for large or long-running work |

## Select Path

| User Intent | Preferred Command | Details |
| --- | --- | --- |
| Create/Update/Delete Managed Agent | `arkcli agent agent ...` | [`references/agent.md`](references/agent.md) |
| Copy an Existing Agent and Rename/Modify Configuration | `arkcli +new-agent --fork <agent-id> [--name <new-name>]` | [`references/agent.md`](references/agent.md#Copy-agent) |
| Choose Available Models for Creating an Agent | `arkcli agent model list` | [`references/agent.md`](references/agent.md#model selection) |
| Choose a Skill for Creating an Agent | Search custom skills only (BytePlus does not expose Market/SkillHub — see [`references/skills.md`](references/skills.md)) | [`references/skills.md`](references/skills.md) |
| Query/Use Custom Skills in the Current Account | First `agent skill list --source custom --limit 100`, continue with `NextPage` if no matches; use `--page-all` or `--skill <skill-id>` for complete candidates | [`references/skills.md`](references/skills.md) |
| Upload a Local Custom Skill Zip | `arkcli agent skill create --zip <file>` or `arkcli agent agent create --skill-zip <file>` | [`references/skills.md`](references/skills.md) |
| Manage Custom Skill Versions or Delete a Skill | List all versions first, then delete in dependency order: non-latest versions, latest version, and finally the Skill | [`references/skills.md`](references/skills.md) |
| Create a Session Environment or Session | `arkcli agent env ...` or `arkcli agent session ...` | [`references/session-files.md`](references/session-files.md) |
| Set an Environment Initialization Script | `arkcli agent env create/update --setup-script @./bootstrap.sh` | [`references/session-files.md`](references/session-files.md#environment-setup-script) |
| Override an Agent or Environment for One Session | `arkcli agent session create --agent-overrides ... --environment-overrides ...` | [`references/session-files.md`](references/session-files.md#session-overrides) |
| Mount a TOS Directory When Creating a Session | After the user provides the path, use `arkcli agent session create --tos-path tos://<bucket>/<prefix>/`; never guess the bucket or prefix | [`references/session-files.md`](references/session-files.md#session-tos-resource) |
| Continue an Existing Session | `arkcli +new session` | [`references/events-chat.md`](references/events-chat.md) |
| Create a New Session and Chat | `arkcli +new session <agent-id> --environment-id <env-id>` | [`references/events-chat.md`](references/events-chat.md) |
| Send Messages to or Stream Replies from a Session | Plain `events send` is write-only; add `--stream` for SSE replies (`--wait` remains a compatibility alias), use `--poll` for long work, or follow with `+tail`. Streaming requests Agent message/thinking deltas by default; use `--no-event-deltas` for complete events only | [`references/events-chat.md`](references/events-chat.md) |
| Diagnose or Export a Session | `arkcli +debug <session-id>` or `arkcli +export <session-id>` | [`references/debug-export.md`](references/debug-export.md) |
| Upload a File to an Existing Session | `arkcli agent session resources add <session-id> --path <file>` | [`references/session-files.md`](references/session-files.md) |
| Query or Upload Files via the Files API | `arkcli agent file upload/list/get/wait/delete` | [`references/session-files.md`](references/session-files.md) |
| Manage Memory Stores or Memories | `arkcli agent memory-store ...` | [`references/interfaces-gaps.md`](references/interfaces-gaps.md) |
| Query MCP/Vault/Credential/MCP OAuth Providers | `arkcli agent vault oauth-provider list` or `arkcli agent vault ...` or `arkcli agent +mcp-login ...` | [`references/mcp-vault.md`](references/mcp-vault.md) |

## Authentication and Profiles

- Before business commands, run `arkcli auth status --format json`. If not logged in, SSO expired, or STS refresh failed, handle login first.
- For data plane Files, Session resources, events, and threads, an ARK API Key is needed. The CLI will prioritize the `api_key` in the profile, or use the global `--api-key` if not set.
- In online environments, use `--env prod` by default; do not default to `stg`.
- Non-interactive SSO logins are two-step: first `arkcli auth login --no-browser` to get a URL; users need to paste the base64 code to complete the login with `arkcli auth login --no-browser --code <code>`.

## Pagination

- Support pagination for lists with `--page-all` global flag; if not explicitly set, the CLI defaults to fetching 100 items per page. The CLI defaults to requesting up to 10 pages, with `--page-limit <N>` to adjust the page size and `--page-delay <ms>` to control the interval.
- Supported: Agent/versions, Env, Session, Skill (custom only — BytePlus does not expose Market/SkillHub), Memory Stores/Memories, Vaults/Credentials/OAuth Providers, Files, Session Events/Threads. The CLI will use the backend pagination mechanisms (`Page`, `PageNumber`, `PageToken`, `after`) and merge the results.
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
| `arkcli agent model list` | Query the whitelist of Managed Agent models; use `--query` to enhance or sort the whitelist with model directory details; the `items[].model` can be used as `--model` |
| `arkcli +new-agent` | Enhanced entry for creating an Agent; supports `--fork/--from` to copy an existing Agent and create a new one |
| `arkcli +iterate` | Update Agent configuration, create a new Session, and run one-shot/REPL; `--environment-id/--env-id` can be omitted, and the latest environment will be chosen automatically |
| `arkcli agent skill search/list/get/create/update/delete/versions/download` | Query custom skills, upload zip files, create/list/download versions, or delete in dependency order: non-latest versions, latest version, then Skill; BytePlus defaults `list/search` to `--source custom` through TOP `ListSkills` |
| `arkcli agent env list/get/create/update/delete` | CRUD for Environment |
| `arkcli agent session list/get/create/update/delete` | CRUD for Session |
| `arkcli agent session resources list/add/get` | CRUD for data plane session resources; `get` is a local filter based on `list` |
| `arkcli agent session events list/send/stream` | Data plane events; streaming requests Event Deltas by default and `--no-event-deltas` falls back to complete events. `user.custom_tool_result` requires `custom_tool_use_id`; `user.tool_result` is self-hosted only |
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
