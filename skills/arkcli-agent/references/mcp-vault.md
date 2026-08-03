# MCP and Vault

## Query MCP to Attach

Registered and usable MCP providers that are registered by the backend can be queried by the following command:

```bash
arkcli agent vault oauth-provider list --limit 100 --format json
```

- The MCP server information in the results can be used as the source for creating or updating an Agent with `--mcp-server`. Do not guess URLs arbitrarily.
- If you want to "attach an MCP to an Agent" such as GitHub, Notion, or WeChat, first check the provider list and filter by the provider's name, URL, and credential type.
- Typically, an additional `mcp_toolset` configuration is needed when attaching an MCP, and `mcp_server_name` should align with the `--mcp-server`'s `name` or `Name`.
- If the provider requires OAuth, use the OAuth MCP login chain to obtain the credential; if it's static bearer, use `agent vault credentials create`.

## Environment Variable Credentials and Token Refresh

Credentials support `Auth.Type=environment_variable`. The typed flags match the frontend form, and sensitive values can be loaded from `@file` to keep them out of shell history:

```bash
arkcli agent vault credentials create <vault-id> \
  --display-name github-token \
  --secret-name GITHUB_TOKEN \
  --secret-value @./github-token.txt \
  --networking-type limited \
  --allowed-host api.github.com \
  --format json
```

This creates `Auth: {Type: environment_variable, SecretName, SecretValue, Networking}`. `--networking-type` must be `limited` or `unrestricted`. Repeat `--allowed-host` for a limited credential; do not supply allowed hosts with unrestricted networking.

Managed MCP OAuth refresh fields can also be supplied directly:

```bash
arkcli agent vault credentials create <vault-id> \
  --auth-type mcp_oauth \
  --mcp-server-url https://example.com/mcp \
  --access-token @./access-token.txt \
  --client-id <client-id> \
  --client-secret @./client-secret.txt \
  --refresh-token @./refresh-token.txt \
  --token-endpoint https://example.com/oauth/token \
  --scope read \
  --format json
```

The CLI writes `Auth.Refresh`; `TokenEndpointAuth.ClientSecret` contains the client secret and the backend manages refresh. `--auth @./credential.json` remains available for a complete object, and explicit typed flags override matching fields in that object.

## OAuth MCP Login

Do not guess the MCP OAuth URL arbitrarily. The correct order is:

1. Check the backend provider:

```bash
arkcli agent vault oauth-provider list --limit 100 --format json
```

2. Only use `arkcli agent +mcp-login` for providers with `CredentialType=mcp_oauth`. Current examples that work include GitHub MCP:

```bash
arkcli agent +mcp-login \
  --vault-id vlt-xxx \
  --name arkcli-github-mcp-oauth-<timestamp> \
  --mcp-server-url https://api.githubcopilot.com/mcp/ \
  --format json
```

3. `arkcli agent +mcp-login` will start a callback server locally, call `CreateVaultOAuthFlow`, print and open the authorization URL, and wait for the backend callback to create the credential. Once successful, it outputs `credential_id`.
4. Use `--no-open` to avoid automatically opening the browser. Manually copy the authorization URL.
5. Afterward, use `agent vault credentials list <vault-id>` to view the created credential. Use `agent vault credentials get` to view non-sensitive fields only if explicitly requested for troubleshooting.

Notes:

- Providers with `CredentialType=static_bearer` are not suitable for `arkcli agent +mcp-login`, and should use `agent vault credentials create` for bearer authentication.
- Do not write user tokens into agent metadata or tags.
- `agent vault credentials get` may display sensitive authentication fields. Unless explicitly requested for troubleshooting, use `agent vault credentials list` instead.
- In the current backend metadata discovery, examples like `https://mcp.notion.so/v1` and `https://mcp.larksuite.com/base` fail. Do not use these as default examples.

## Attach MCP to an Agent

- Only attach an MCP when explicitly provided by the user or contextually required. Otherwise, use `agent vault oauth-provider list` to check available providers.
- Typically, both `McpServers` and `mcp_toolset` need to be configured when attaching an MCP.
- Note: `--tool` replaces the entire `Tools` array. If you need to preserve default `agent_toolset_20260701` and `mcp_toolset`, include the complete default `Configs` array in the same array. Do not use only `mcp_toolset` or `{type: agent_toolset_20260701}`.
- If unsure about the default tool structure, refer to `references/agent.md#default-agent-tools`. First run the same `agent agent create ... --dry-run` command and inspect the complete default `Configs` plus the added `mcp_toolset` in `preview.v1`; create the Agent only after explicit confirmation.
- The example `tools-with-default-agent-toolset-and-github-mcp.json` is not an internal file; it represents a custom `Tools` array file prepared by the caller.

```bash
arkcli agent agent create \
  --name arkcli-mcp-agent \
  --model <items[].model from agent model list> \
  --mcp-server '{type: url, name: github, url: https://api.githubcopilot.com/mcp/}' \
  --tool @tools-with-default-agent-toolset-and-github-mcp.json
```

- `MCPConnectionFailed` usually indicates issues with the URL, protocol, authentication, or `tools/list` initialization, not necessarily an Agent creation failure.

## Example

```bash
arkcli agent vault oauth-provider list --limit 100 --format json
arkcli agent vault create --display-name arkcli-mcp-login-vault-20260706 --format json
arkcli agent +mcp-login --vault-id vlt-xxx --name arkcli-github-mcp-oauth-20260706 --mcp-server-url https://api.githubcopilot.com/mcp/ --format json
```
