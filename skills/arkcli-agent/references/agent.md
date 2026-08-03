# Agent

## Create Agent SOP

User says"Create a XXX agent / Agent"Execute the link when this occurs,Don't just concatenate one `agent create`.

1. Run `arkcli auth status --format json`. Confirm login, profile, project, and
   API key, then read `byteplus_sso.identity.verified`. If it is `false`, stop
   before model lookup and send the user to
   `https://console.byteplus.com/user/basics/`; both account-opening and
   payment verification must be complete. If the field is absent, continue
   without guessing and preserve any structured backend error. See
   [`realname-gate.md`](../../arkcli-auth/references/realname-gate.md).
2. If the user does not provide an exact model,Enhance model whitelist based on user intent first:`arkcli agent model list --query "<user intent/Domain>" --primary-only --format json`.it does BytePlus ModelArk None `agent_support=true` of Managed Agent whitelist for candidate set,use model directory again `/models search` capability of, context, Modal, Order Signal Enhancement `detail` Sort hits by matching fields first;returning `items[].model` As is `--model`.
3. extend user intent skill Select context.example:data analysis -> `data analysis Excel CSV table BI SQL`;Code Assistant -> `code programming repo bash`;document generation -> `documentation Write summary Markdown`.
4. Create Agent default prefers existing account ones custom skill Choose Ark,even the user hasn't explicitly said"Using custom skill":On-demand Execution `arkcli agent skill list --source custom --limit 100 --format json`.by BytePlus ModelArk Read this page fully `Items`,By Name, Description, Capability and version check;No suitable candidates,Reacting `NextPage` As-is Pass Through `--page` Scroll down to continue,Until hit or no next page.don't just call `search --source custom "<query>"` Subsequent first;Users request a full list explicitly or need it offline for analysis. `--page-all`.
5. custom skill Paginated search still no suitable candidates: BytePlus does not support market/SkillHub skill sources. If no custom skill matches, upload a local skill zip via `arkcli agent skill create --zip <path>` or create a base agent without a skill and explain no matching custom skill.
6. Assembly parameters:Default speed `standard`,Patch Field Resolution system prompt,Tag related skill.Online Test Resource Name Usage `arkcli` Prefix;Nameless by default `arkcli-<domain>-agent-<YYYYMMDDHHMMSS>`,Such `arkcli-data-agent-20260707153000`.
7. First run the same `agent agent create ... --dry-run --format json` command and read its zero-network `preview.v1` plan. Restate `Model`, `Skills`, default `Tools`, `McpServers`, and every `unresolved` item. An unresolved item means real execution still has to check Managed Agent/model activation, resolve the latest version of a bare custom Skill, or upload `--skill-zip`; never describe it as already validated. After explicit confirmation, remove `--dry-run` and run the real create command.
8. Real creation time,CLI it will first check BytePlus ModelArk Capability and model enabled status:Model not activated will go through shared model activation confirmation link;Non-interactive environment will not automatically activate,Returned `model_activation_required`.BytePlus ModelArk Product/Ability not enabled,TTY This will prompt the user to confirm and call the front-end equivalent. `OpenChargeItems(ResourceType=DataManagedAgentSum, ResourceNames=[sandbox, web_search])`,Non-interactive Environment Return `managed_agent_activation_required`,Does not automatically activate.
   - If explicit confirmation from the user is already obtained in the conversation,Non-interactive call can retry the original command and add `--yes` from environment variables `ARKCLI_ALLOW_HEADLESS_ACTIVATION=1`.do not set this environment variable without user confirmation.
9. Real creation upon creation `agent agent get <agent-id> --format json` Confirm depot slot.Server final configuration should be displayed when echoing to users.,Do not display only"Created"or summarize a single field.
10. User requests end-to-end validation,Create again env/session,Send a minimal message,Pull events/thread/resources.unless the user explicitly requests to clean,Do not delete created resources.

model candidate, skill Candidates, MCP provider Candidate Independence;Create in parallel first:

```bash
arkcli agent model list --primary-only --format json
arkcli agent model list --query "<capability-query>" --primary-only --format json
arkcli agent skill list --source custom --limit 100 --format json
arkcli agent vault oauth-provider list --limit 100 --format json
```

## model selection

`agent agent create --model` Model must be specified exactly ID.Don't name bare models or displays by impression..

Preferred source is searched by user intent:

```bash
arkcli agent model list --query "data analysis Excel CSV SQL agent" --primary-only --format json
```

`--query` mode still remains BytePlus ModelArk Whitelist as Master Table,Reverse generate candidates from model directory.It will call `models search` Get details and enhance whitelist model:Matched whitelist models are brought `detail` Fields are placed in front.;Whitelisted models that miss the target are retained.,Only exists `detail`.`detail` Field contains signals for assessing fit,such as `display_name`, `description`, `context_window`, `input_modalities`, `output_modalities`, `capabilities`, `lifecycle_status`.

Only want to list whitelists available:

```bash
arkcli agent model list --primary-only --format json
```

Select rule:

- default only from `agent_support=true` select from the result;`agent model list` default filtering non Agent model.
- Primary Selection `primary_version=true` entry;Do not concatenate across entries for named versions with the same name.
- Return during creation `model` Field,such as `items[0].model`,Do not return `id`.
- User explicitly requests a model family,using `--name <keyword>` Reduce BytePlus ModelArk whitelist scope;Prioritize natural language intent when present `--query`:

```bash
arkcli agent model list --query "Complex Inference Agent" --name doubao-seed-2-0-pro --primary-only --format json
```

- Need troubleshooting or confirmation why a model is not selectable,Add `--include-all`,View `agent_support=false` entry.
- Use structured filtering with hard metrics:`--min-context-window 200000`, `--capability thinking`, `--capability functioncall`, `--multimodal`, `--input-modality text,image`, `--output-modality text`, `--strict-filter`.
- User has granted full model ID it can directly use,but if creation fails due to unsupported model / Nonexistent,Back to `agent model list` reselect.

Model value example:

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

## script output

`+new-agent` output compatible `data.agent` Structure,Script-friendly:

```bash
arkcli +new-agent --fork agent-xxx --format json --transform "data.agent.id"
arkcli +new-agent --fork agent-xxx --format yaml > new-agent.yaml
arkcli +new-agent --fork agent-xxx --no-echo --format json
```## Default Agent Tools

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
- When needing `advisor` and retaining default tools, the full array must be explicitly provided, e.g., `--tool '[{Type: agent_toolset_20260701, Name: agent_toolset_20260701, Configs: [{Name: advisor, Enabled: true}], PermissionPolicy: {Type: always_allow}}]'`.
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
