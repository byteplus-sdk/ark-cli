# Agent

When configuring `speed/thinking/reasoning_effort/service_tier`, query the target model's [runtime parameter metadata](model-config.md). Values from another model are not universal enums.

## Create Agent SOP

User says"Create a XXX agent / Agent"Execute the link when this occurs,Don't just concatenate one `agent create`.

1. Run `arkcli auth status --format json`. Confirm login, profile, project, and
   API key, then read `byteplus_sso.identity.verified`. If it is `false`, stop
   before model lookup and send the user to
   `https://console.byteplus.com/user/basics/`; both account-opening and
   payment verification must be complete. If the field is absent, continue
   without guessing and preserve any structured backend error. See
   [`realname-gate.md`](../../arkcli-auth/references/realname-gate.md).
2. If the user does not provide an exact model, list all candidates with `arkcli agent model list --format json`. Only for explicit context/modality/capability requirements, fetch and filter by [exact-version metadata](#exact-version-filtering). Otherwise do not make per-model metadata requests. Use the selected `items[].model` unchanged as `--model`.
3. extend user intent skill Select context.example:data analysis -> `data analysis Excel CSV table BI SQL`;Code Assistant -> `code programming repo bash`;document generation -> `documentation Write summary Markdown`.
4. Create Agent default prefers existing account ones custom skill Choose Ark,even the user hasn't explicitly said"Using custom skill":On-demand Execution `arkcli agent skill list --source custom --limit 100 --format json`.by BytePlus ModelArk Read this page fully `Items`,By Name, Description, Capability and version check;No suitable candidates,Reacting `NextPage` As-is Pass Through `--page` Scroll down to continue,Until hit or no next page.don't just call `search --source custom "<query>"` Subsequent first;Users request a full list explicitly or need it offline for analysis. `--page-all`.
5. If no custom skill matches, BytePlus can query read-only Ark skills with `agent skill list --source ark`; it does not support market/SkillHub. Otherwise upload a local skill zip with `arkcli agent skill create --zip <path>` or create a base Agent and explain that no matching skill was found.
6. Assemble parameters and a domain-specific system prompt. Use `--thinking`, `--reasoning-effort`, and `--service-tier` only when required by the user and supported by server `ModelMeta`. Do not inject legacy `speed=standard`, and never map `speed` to the new fields. Use an `arkcli` prefix for online test resources; if unnamed, generate `arkcli-<domain>-agent-<YYYYMMDDHHMMSS>`.
7. First run the same `agent agent create ... --dry-run --format json` command and read its zero-network `preview.v1` plan. Restate `DisplayName`, `Model`, `Skills`, default `Tools`, `McpServers`, and every `unresolved` item. A bare custom Skill intentionally omits `Version` so the service always selects latest; it does not require online version resolution. Managed Agent/model activation checks and `--skill-zip` upload remain unresolved until real execution. After explicit confirmation, remove `--dry-run` and run the real create command.
8. Real creation time,CLI it will first check BytePlus ModelArk Capability and model enabled status:Model not activated will go through shared model activation confirmation link;Non-interactive environment will not automatically activate,Returned `model_activation_required`.BytePlus ModelArk Product/Ability not enabled,TTY This will prompt the user to confirm and call the front-end equivalent. `OpenChargeItems(ResourceType=DataManagedAgentSum, ResourceNames=[sandbox, web_search])`,Non-interactive Environment Return `managed_agent_activation_required`,Does not automatically activate.
   - If explicit confirmation from the user is already obtained in the conversation,Non-interactive call can retry the original command and add `--yes` from environment variables `ARKCLI_ALLOW_HEADLESS_ACTIVATION=1`.do not set this environment variable without user confirmation.
9. Real creation upon creation `agent agent get <agent-id> --format json` Confirm depot slot.Server final configuration should be displayed when echoing to users.,Do not display only"Created"or summarize a single field.
10. User requests end-to-end validation,Create again env/session,Send a minimal message,Pull events/thread/resources.unless the user explicitly requests to clean,Do not delete created resources.

model candidate, skill Candidates, MCP provider Candidate Independence;Create in parallel first:

```bash
arkcli agent model list --format json
arkcli agent model list --query "<capability-query>" --format json
arkcli agent skill list --source custom --limit 100 --format json
arkcli agent vault oauth-provider list --limit 100 --format json
```

`--name` is the stable resource name and `--display-name` is the user-visible label. Model runtime fields may be passed inside `--model` or as dedicated flags; dedicated flags take precedence:

```bash
arkcli agent agent create \
  --name arkcli-reasoning-agent-20260828 \
  --display-name "Reasoning Assistant" \
  --model '{id: <items[].model>, thinking: enabled, reasoning_effort: high, service_tier: auto}' \
  --reasoning-effort medium \
  --system @./system.md \
  --format json
```

- The model object accepts `id`, `provider`, `speed`, `thinking`, `reasoning_effort`, and `service_tier`, including camel/Pascal aliases.
- `speed` is legacy-only. When omitted, the CLI leaves it absent.
- `agent agent update <id> --display-name ""` clears the display name.
- Normal updates omit `--version`, for example `arkcli agent agent update <id> --description "Updated description"`. The CLI calls `GetAgent` and supplies its current `Result.Version` to `UpdateAgent`; do not ask users to look up and enter a version manually. Model-ID hydration and Metadata merging reuse the same read.
- Optional `--version <positive-integer>` pins a previously read version for concurrency checking. `Version` in `--file`/stdin is also supported; an explicit flag wins. Omitted or `0` means automatic lookup; negative versions are invalid. A failed lookup or missing valid version prevents the write. Surface subsequent version conflicts without automatically refreshing the version and retrying the write.
- With no version, zero-network `--dry-run` shows `GetAgent -> UpdateAgent` with `<current-agent-version>` and `unresolved`; this is not a real or reserved version. An explicit version with no model/Metadata lookup produces only `UpdateAgent`. This rule applies only to Agent update, not Session version parameters.
- Runtime-only model updates do not require the model ID again. During real execution, the CLI reads the current Agent first and supplies its current `Model.Id` to `UpdateAgent`.
- An explicit `Model.Id` from `--model <new-id>`, a model object, or `--file` always wins; the CLI never replaces it with the old ID. Dedicated `--thinking`, `--reasoning-effort`, `--service-tier`, and `--speed` flags still override the same fields in the model object.
- A `--model` object without an ID is treated as a runtime-settings patch and hydrated with the current `Model.Id`. The CLI fails before the write if the current Agent does not return a model ID.
- `agent agent update ... --dry-run` remains zero-network. When the ID must be hydrated, preview shows `GetAgent -> UpdateAgent` and represents the execution-time value with `<current-agent-model-id>` plus `unresolved`; never send the placeholder as a real ID.
- `agent agent list --display-name <keyword>` filters on the display label; table output keeps `DISPLAY_NAME` separate from stable `NAME`.

## model selection

`agent agent create --model` Model must be specified exactly ID.Don't name bare models or displays by impression..

List all eligible candidates by default:

```bash
arkcli agent model list --format json
```

`--query` mode still remains BytePlus ModelArk Whitelist as Master Table,Reverse generate candidates from model directory.It will call `models search` Get details and enhance whitelist model:Matched whitelist models are brought `detail` Fields are placed in front.;Whitelisted models that miss the target are retained.,Only exists `detail`.`detail` Field contains signals for assessing fit,such as `display_name`, `description`, `context_window`, `input_modalities`, `output_modalities`, `capabilities`, `lifecycle_status`.

Optionally enrich details and rank by intent; this is not a default step:

```bash
arkcli agent model list --query "data analysis Excel CSV SQL agent" --format json
```

Select rule:

- default only from `agent_support=true` select from the result;`agent model list` default filtering non Agent model.
- Neither Agent nor tool discovery defaults to `--primary-only` or locally excludes `primary_version=false`. Filter to primary versions only when the user explicitly asks. Primary does not mean latest; never combine fields from different version entries.
- Return during creation `model` Field,such as `items[0].model`,Do not return `id`.
- If the user explicitly requests a model family, narrow candidates with `--name <keyword>`:

```bash
arkcli agent model list --name doubao-seed-2-0-pro --format json
```

- Need troubleshooting or confirmation why a model is not selectable,Add `--include-all`,View `agent_support=false` entry.
- For explicit hard requirements, filter by [exact-version metadata](#exact-version-filtering). List does not support `--size`, `--modality`, `--input-modality`, `--output-modality`, `--multimodal`, `--min-context-window`, `--capability`, or `--strict-filter`. Generic `models search` flags remain unchanged.
- User has granted full model ID it can directly use,but if creation fails due to unsupported model / Nonexistent,Back to `agent model list` reselect.

### Exact-version filtering

Use this flow only for explicit requirements such as at least 200000 context tokens, image input, or function calling:

1. List all eligible candidates with `agent model list` (`--usage tool` for tool models). Preserve each `name`, `version`, and `model`. Apply an explicit name constraint with `--name`; do not default to primary versions.
2. Reuse metadata already retrieved for the same name and version. Otherwise query `ListModelMetaDatas` as needed. `models get <name> --version <version>` provides product details; raw metadata has no dedicated product command, so use [API Explorer](../../arkcli-api-explorer/SKILL.md). Confirm the registry and local preview before the real read:

```bash
arkcli api model.list_model_meta_datas \
  --params '{"FoundationModelName":"<items[].name>","FoundationModelVersion":"<items[].version>","Keys":["context_window"],"PageNum":1,"PageSize":1000}' \
  --format json
```

Replace placeholders with real candidate fields. Request only relevant keys; batch multiple keys for the same version in one call:

| Requirement | Metadata key / interpretation |
| --- | --- |
| At least N context tokens | `context_window`: parse `Data[0]` as a positive integer and compare with N |
| Input/output modalities | `input_modalities` / `output_modalities`: inspect actual text/image/video/audio values in `Data` |
| Thinking / function calling / MCP | `thinking.support` / `functioncall.support` / `mcp.support`: read actual support values rather than inferring from names |

3. Read `ModelMetaDatas` or `Result.ModelMetaDatas` from the actual response and verify its model name/version. If pagination indicates more rows, increment `PageNum`; do not rely on `--page-all`. A metadata row's `Version` is an edit revision, not `FoundationModelVersion`.
4. Filter with matching-version data only. Missing, empty, unparseable, or failed metadata is unknown, not zero/unsupported and not verified compliance. Distinguish matching, non-matching, and unknown candidates. For strict requirements recommend only verified matches and disclose unverified entries; multiple matches still require user selection.

Query-mode `detail` is joined by name and may describe a primary version, not another version's hard limits. `ListModelMetaDatas` is also distinct from `ListManagedAgentModelMetaDatas`, used by `agent model config` for runtime parameter values. Tool eligibility still comes only from version-level ArkModels `epa_tool_model.support`. Do not add metadata calls without explicit requirements or turn selection guidance into a write gate for user-supplied model IDs.

### Model value example

```bash
arkcli agent agent create \
  --name arkcli-data-analysis-agent-20260706 \
  --model <items[].model from agent model list> \
  --system "You are a data analysis agent. Help users inspect datasets, reason about metrics, write analysis code, and summarize findings clearly." \
  --format json
```

## Copy Agent

User says"Copy / fork / Based on existing agent Update to a new version"Timestamp,Prioritize Using `+new-agent --fork`,Avoid manual get reassemble fully create Request.

```bash
arkcli +new-agent --fork agent-xxx --format json
```

- User explicitly granted the source `agent-id` but without a new name,Don't stop at just names.;CLI Yes, first. `GetAgent`,default source Agent of `Name` Add `copy-` Prefix,Immediately `copy-<source-agent-name>`.If source name empty,Just fallback to `copy-<agent-id-tail>`.
- If the user says"Copy that data analysis agent"but without it ID,Use first `agent agent list --name <keyword>` Choose candidates;continue replication if there are any candidates,Candidates must be confirmed by the user when multiple are selected `agent-id`.
- `+new-agent --fork` must read the source Agent online and cannot provide a local Client Preview, so it does not register `--dry-run`. Inspect the source with `agent agent get <id>`, restate overrides, obtain confirmation, and then execute the copy.
- `--fork` / `--from` Yes, first. `GetAgent`,Copy Source Agent of `Model`, `System`, `Description`, `Tools`, `Skills`, `McpServers`, `Multiagent`, `Metadata`, `Tags`,Invoke again `CreateAgent` Create new Agent.
- `--name` explicitly override default copy name.
- User-provided create flags overrides copied configuration,such as `--system`, `--model`, `--description`, `--skill`, `--tool`, `--mcp-server`.
- `--skill` / `--tool` / `--mcp-server` replace the corresponding list rather than append. Use read-only `agent agent get` to inspect the source, then pass the complete list.
- default creates again after success `GetAgent` Echo Final Configuration;Want only to take `CreateAgent` Usage `--no-echo`.
- Result Creation `System` actual effect system prompt.Human-readable echo must display the following fields at a minimum:
  - Identity:`Id`, `Name`, `Description`, `Version`, `ProjectName`
  - model:`Model.Id`, `Model.Speed` and other model configurations returned by the server
  - Agent Behavior:Whole `System`, `Tools`, `Skills`, `McpServers`
  - Expanded configuration:`Multiagent`, `Metadata`, `Tags`
  - server response timestamp:`CreateTime`/`CreatedAt`, `UpdateTime`/`UpdatedAt`
- Structure Output Use `agent agent get <agent-id> --format json` or `--format yaml` Keep all non-empty fields returned by the server;The caller shall not discard, truncating or replacing config field with a summary.human-readable summary compresses time, ID Wait for display format,But this cannot hide the above configuration content..
- If submitted `request.System` Non-empty but create response or `GetAgent.Result.System` empty,Critical Reporting"Server did not echo/Uncommitted",Do not assume prompt Effective,and retain request values and server values for troubleshooting.
- `+new-agent` Current disabled LLM Draft and template;Natural language understanding, parameter selection, User confirms by calling `arkcli` of BytePlus ModelArk AI agent Complete.

## Iteration Agent and create Session

User says"Change this agent Then try it out / Update system run again / Adjust tools New after session verify"Timestamp,using `+iterate`,Don't manually run three or four commands at a time..

`+iterate` Occurs:

1. `GetAgent` Show current version.
2. Updates were provided flag,then use the current version `UpdateAgent`.
3. Call `CreateSession` Start a new one. session.
4. Has `--message` Send the first message and stream output;TTY And there is `--message` Enter time `+new session` REPL;Non TTY None message output structured result only.

```bash
arkcli +iterate agent-xxx \
  --system @./prompts/da-v2.md \
  --message "Use the new configuration to explain how you're going to analyze it. sales.csv" \
  --format json
```

- `--environment-id` / `--env-id` Optional;Omit CLI using the most recently created project in the current project Environment.Should be Explicitly Passed When Explicitly Required by Environment.
- Don't assume the environment is provided by the user ID interrupt or request supplementary information:Actual Execution Time CLI Uses `ListEnvironments` By `CreateTime Desc, Limit=1` Automatic selection of latest environment;Only when there are no available environments,Then prompt to create an environment or explicitly provide one `--environment-id`.
- `--diff` does not call `ListEnvironments` and shows `<auto-select-latest-environment>` in `CreateSession.EnvironmentId`. This is a placeholder. `+iterate` does not register Client Preview `--dry-run`.
- `--resource`, `--vault-id` / `--vault-ids`, `--tags` It will be passed to the new session.
- `--diff` reads the current Agent online and previews `UpdateAgent`, `CreateSession`, and chat requests without remote mutation. It is a command-specific online diff, not zero-network Client Preview.
- `--no-chat` Update Only agent and create session,Do not send message, No Input REPL.
- `--tool`, `--skill`, and `--mcp-server` replace the corresponding configuration with the complete semantic value supplied by the user.
- When `+iterate` changes only `--thinking`, `--reasoning-effort`, `--service-tier`, or `--speed`, it reuses the current `Model.Id` from the Agent it already read; an explicit new model ID still wins.
- `--display-name`, including an empty string used to clear it, is an Agent update and must not be skipped as “no update.”
- With `--agent-overrides`, the Session Agent version belongs in `AgentWithOverrides.Version`; the final CreateSession request must not also contain the mutually exclusive top-level `AgentId` or `AgentVersion`.

## script output

`+new-agent` output compatible `data.agent` Structure,Script-friendly:

```bash
arkcli +new-agent --fork agent-xxx --format json --transform "data.agent.id"
arkcli +new-agent --fork agent-xxx --format yaml > new-agent.yaml
arkcli +new-agent --fork agent-xxx --no-echo --format json
```

## Custom tools

`agent agent create` / `+new-agent` accept custom tools through `--tool`; updates use the same structure. A custom tool is an entry in `Tools` with `Type: custom`, not a separate `custom_tools` field, `--custom-tools` flag, or custom Skill.

| Parameter | Type / requirement | Meaning |
| --- | --- | --- |
| `Type` | string, required: `custom` | Custom tool type |
| `Name` | string, required | Model-visible tool name; the current backend allows 1–128 ASCII letters, digits, underscores, or hyphens, with unique custom tool names per Agent |
| `Description` | string, required | Purpose, usage context, and behavior; the current backend limit is 10,000 characters |
| `InputSchema` | object, required for custom tools | Input schema; supply an object, not a JSON string or base64 |
| `InputSchema.Type` | string, optional | `object`; the backend defaults to `object` when omitted. Prefer writing it explicitly |
| `InputSchema.Properties` | object, optional | Parameter names mapped to JSON Schema, e.g. `city: {type: string}`. Parameter names and nested schema keys are preserved, not converted to PascalCase |
| `InputSchema.Required` | string[], optional | Required parameter names matching Properties keys, e.g. `[city]`; omit or use `[]` when none are required |

The current backend allows up to 8 custom tools per Agent. Business validation, including names and counts, remains server-owned. Custom tools do not use toolset fields `Configs`, `DefaultConfig`, or `McpServerName`.

This example configures only one custom tool, without default tools. Replace the model placeholder with the selected available ID:

```bash
arkcli agent agent create \
  --name arkcli-weather-agent \
  --model '<selected-model-id>' \
  --tool '{
    Type: custom,
    Name: get_weather,
    Description: Get weather for a city,
    InputSchema: {
      Type: object,
      Properties: {city: {type: string, description: City name}},
      Required: [city]
    }
  }' \
  --dry-run --format json
```

- `--tool` accepts a JSON/YAML object, array, or repeated flags. Tool fields accept PascalCase / camelCase / snake_case aliases, such as `InputSchema` / `inputSchema` / `input_schema`. For complete file/stdin requests, use TOP fields with `Tools: [...]` and the structure above.
- All supplied entries form the complete replacement `Tools` array; defaults are not appended. To keep default tools, include the `agent_toolset_20260701` toolset alongside custom entries. Before appending through an update, get the current configuration and submit the full list. `--tool '[]'` clears tools.
- Check that dry-run shows an object InputSchema and unchanged Properties, then create and read back with get. A declaration does not upload an executable implementation or make the CLI automatically execute arbitrary functions. Return results through the [custom tool result event](events-chat.md), using the corresponding `custom_tool_use_id`.

## Tool model bindings (supported tools only)

When creating an Agent, enter this discovery/configuration flow only if the user explicitly requests multimodal tools (such as image or video generation) or supplies such a tool configuration. Otherwise do not query `agent model list --usage tool` or add `agent_toolset_multimodal_20260825`, `image_generate`, `video_generate`, or their `Configs[].Models`. Multimodal model inputs, available tool-model candidates, or example fields do not imply a request for generation tools. Retain the ordinary default toolset when tools are unspecified; do not pass `--tool '[]'` to clear it. Preserve explicit user configurations rather than silently removing them under this rule.

`Tools[].Configs[].Models` is not a universal tool parameter or the Agent's primary `Model`. The current backend contract supports it only for `image_generate` and `video_generate` within `Type: agent_toolset_multimodal_20260825`. Do not add it to ordinary tools, custom tools, MCP, or `DefaultConfig`. The CLI does not inject models or maintain a tool allowlist; the service validates availability, model permissions, and modality compatibility.

Each entry in `Models` accepts:

| Field | Type / requirement | Meaning |
| --- | --- | --- |
| `ID` | string, required | Exact model or Endpoint ID; the sole model identity. The TOP field is uppercase `ID` |
| `Default` | boolean, optional | Preferred binding for this tool configuration; explicit `false` is preserved and omission does not trigger a CLI default |
| `Name` | string, optional | Display name only; cannot replace the ID |
| `Version` | string, optional | Display version only; not model identity or the Agent version |

The current service allows at most 10 bindings per configuration with distinct IDs. Multiple bindings require exactly one `Default: true`; a single binding may omit it. The Agent primary-model whitelist is not an image/video-generation model catalog. Query tool usage below, then confirm the target tool's modality and permissions.

### Discover tool models and cache behavior

```bash
arkcli agent model list --usage tool --format json
arkcli agent model list --usage tool --name <model-name-keyword> --format json
arkcli agent model list --usage tool --include-all --format json
arkcli agent model list --usage tool --refresh-cache --format json
```

| Parameter/output | Meaning |
| --- | --- |
| `--usage agent` | Default: primary Agent models, using version-level `ep_agent.support`; preserves existing behavior. |
| `--usage tool` | Tool models, using version-level `epa_tool_model.support`; does not require `ep_agent.support` too. Other usage values are rejected. |
| `--include-all` | Skip the selected usage's support filter for diagnosis; not every returned model is eligible. |
| `--primary-only` | Locally keep server-designated primary versions from the full cache. Omit by default for both Agent and tool discovery; use only when the user explicitly requests primary versions, so eligible non-primary versions are not lost. |
| `--name` | Local case-insensitive substring match against `name` or versioned `model`. |
| `--query` | Optional detail enrichment/ranking with the selected usage's ArkModels candidates as the source of truth. Unmatched candidates remain without truncation. Details are joined by name, not authoritative for a specific version's hard requirements. |
| `items[].tool_model_support` | Version-level tool eligibility, separate from `agent_support`. Metadata keys are case-insensitive; any Data value equal to true/1/yes/y after trimming and case folding indicates support. |
| `items[].model` | Exact versioned identifier to put in `Tools[].Configs[].Models[].ID`, not the catalog's internal `id` (such as epm-*). |
| `--refresh-cache` | Synchronously refresh ArkModels without requiring query; failure cooldown still applies. Query mode also refreshes existing search caches. |
| `cache` | CLI returns `source` (network/cache), `fetched_at`, and `expires_at` for candidate provenance. |

The CLI persists the full ArkModels catalog for 5 minutes, isolated by product, environment, account/user, profile, region, and project. Agent/tool usage and name/primary filters share the same full-version snapshot. Ordinary fresh-cache reads do not request ArkModels again; expiry refreshes synchronously and a cross-process lock coalesces overlapping refreshes. Cache access failures are reported, not bypassed with repeated API requests. Request failures retain the previous snapshot and enforce a 1-minute cooldown, without presenting expired data as current eligibility. Retry after cooldown. Query detail enrichment uses its existing separate caches and is not covered by the candidate-cache hit guarantee.

Tool eligibility comes only from ArkModels. Do not substitute `agent model config --key epa_tool_model.support` (ListManagedAgentModelMetaDatas) or name-collapsed primary-version search data. Empty candidates mean the current catalog declares none, not that all tools reject Models. Eligibility does not guarantee tool modality compatibility, account permissions, activation, or successful inference. The CLI does not add the frontend's trial-display tag restriction. Ask the user to choose among multiple candidates; do not automatically bind a model. An explicitly supplied ID can still be submitted directly to create/update: cache contents do not introduce a new write-time gate.

Supply this fragment through create/update `--tool`, or as an entry in a complete JSON/YAML request's `Tools` array. Replace placeholders with actual model IDs:

```yaml
Type: agent_toolset_multimodal_20260825
Configs:
  - Name: image_generate
    Enabled: true
    Models:
      - ID: <selected-image-model-id>
        Default: true
  - Name: video_generate
    Enabled: true
    Models:
      - ID: <selected-video-model-id>
```

`--tool` still replaces the complete list: get the existing Agent first, retain other required tools/configurations, and submit the full `Tools` list. Client Preview checks local structure, not backend acceptance. Get/list/versions output and `+new-agent --fork` preserve tool bindings; verify `Configs[].Models`, not just the primary `Model.Id`.

## Default Agent Tools

When creating an agent without explicit `--tool` and without `Tools` in the request body, the CLI automatically adds the default `Tools` array:

```yaml
Type: agent_toolset_20260701
Name: agent_toolset_20260701
Configs:
  - Name: bash
    Enabled: true
    PermissionPolicy: { Type: always_allow }
  - Name: read
    Enabled: true
    PermissionPolicy: { Type: always_allow }
  - Name: write
    Enabled: true
    PermissionPolicy: { Type: always_allow }
  - Name: edit
    Enabled: true
    PermissionPolicy: { Type: always_allow }
  - Name: glob
    Enabled: true
    PermissionPolicy: { Type: always_allow }
  - Name: grep
    Enabled: true
    PermissionPolicy: { Type: always_allow }
  - Name: web_fetch
    Enabled: true
    PermissionPolicy: { Type: always_allow }
  - Name: web_search
    Enabled: true
    PermissionPolicy: { Type: always_allow }
DefaultConfig:
  Enabled: true
  PermissionPolicy: { Type: always_allow }
```

- Explicitly passing `--tool '[]'` indicates the default tools are to be disabled, which must be respected.
- When providing any `--tool`, the passed value is the complete `Tools` array. The CLI does not append or merge default tools.
- When needing `advisor` and retaining default tools, provide both toolsets in the complete array, e.g., `--tool '[{Type: agent_toolset_20260701, Name: agent_toolset_20260701}, {Type: evolution, Configs: [{Name: advisor, Enabled: true}]}]'`. `advisor` belongs to the separate `evolution` toolset, not `agent_toolset_20260701`; do not attach a permission policy to it. Availability is subject to backend account/project enablement.
- Do not manually write default tools unless the user wants to adjust permissions or explicitly disable them.

### Modify Default Tool Permissions

When a user says, "set the policy of `<tool-name>` to always ask / always confirm," do not modify just one config. Since `--tool` is a complete array, it must be replaced with the full default toolset, and only the `PermissionPolicy.Type` for the specific tool must be changed.

Permission policy values:

- Always allow: `always_allow`
- Always ask: `always_ask`

For example, "create a data analysis agent with the `write` policy set to always ask":

```bash
arkcli agent agent create \
  --name arkcli-data-analysis-agent-20260706 \
  --model <items[].model from agent model list> \
  --system "You are a data analysis agent. Help users inspect datasets, reason about metrics, write analysis code, and summarize findings clearly." \
  --skill skill-xxx \
  --tool '[{
    Type: agent_toolset_20260701,
    Name: agent_toolset_20260701,
    DefaultConfig: {Enabled: true, PermissionPolicy: {Type: always_allow}},
    Configs: [
      {Name: bash, Enabled: true, PermissionPolicy: {Type: always_allow}},
      {Name: read, Enabled: true, PermissionPolicy: {Type: always_allow}},
      {Name: write, Enabled: true, PermissionPolicy: {Type: always_ask}},
      {Name: edit, Enabled: true, PermissionPolicy: {Type: always_allow}},
      {Name: glob, Enabled: true, PermissionPolicy: {Type: always_allow}},
      {Name: grep, Enabled: true, PermissionPolicy: {Type: always_allow}},
      {Name: web_fetch, Enabled: true, PermissionPolicy: {Type: always_allow}},
      {Name: web_search, Enabled: true, PermissionPolicy: {Type: always_allow}}
    ]
  }]' \
  --format json
```

If the user requests different policies for multiple tools, modify them in the same `agent_toolset_20260701` `Configs` array, do not split into multiple `agent_toolset` entries.

## Example

```bash
# Agent + Skill
arkcli agent skill search "Excel data analysis" --source custom --limit 10 --format json
arkcli agent agent create \
  --name arkcli-data-analysis-agent-20260706 \
  --model <items[].model from agent model list> \
  --system "You are a data analysis agent. Help users inspect datasets, reason about metrics, write analysis code, and summarize findings clearly." \
  --skill skill-xxx \
  --format json

# Fork existing agent
arkcli +new-agent --fork agent-20260707063932-vbfjd --format json
```
