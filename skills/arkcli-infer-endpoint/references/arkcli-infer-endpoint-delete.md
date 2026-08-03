# arkcli infer endpoint delete

Delete an inference endpoint (irreversible)

## Usage

```bash
arkcli infer endpoint delete <endpoint-id> [flags]
```

## Arguments

| Argument | Description | Required |
|----------|-------------|----------|
| `<endpoint-id>` | The ID of the endpoint to delete | Yes |

## Flags

| Flag | Type | Description | Required |
|------|------|-------------|----------|
| `--dry-run` | bool | Emit the local `DeleteEndpoint` Client Preview without reading or deleting the endpoint | No |
| `--yes` | bool | Skip the interactive Y/N prompt (TTY only). **Non-interactive mode: this flag does NOT authorize deletion** — set `ARKCLI_ALLOW_HEADLESS_DELETE=1` instead. | No |
| `-h`, `--help` | | help for delete | No |

## Confirmation mechanism (high risk, must read)

Deletion is irreversible. `infer endpoint delete` uses a **three-branch confirmation gate**:

| Environment | Behavior |
|------|------|
| Interactive terminal (TTY) | Shows a strong warning + `[y/N]`. The command executes only when the user enters `y`. `--yes` skips the prompt and counts as confirmation. |
| Non-interactive (agent / CI / pipeline) + `ARKCLI_ALLOW_HEADLESS_DELETE=1` | Allows the operation after printing an audit line (for true unattended automation). |
| Non-interactive + env not set | **hard-refuse** (`requires_headless_gate`). `--yes` is ignored here. |

Design intent: any agent can reflexively add `--yes`, but an agent's stdin is not a real TTY. Therefore, "real TTY or explicit env" is a hard gate that the agent cannot fabricate, preventing agents from silently approving deletion on behalf of the user with `--yes`. This aligns with the `ARKCLI_ALLOW_HEADLESS_ACTIVATION` pattern enabled for `infer endpoint create`.

### Notes for agent calls

When agents (Claude Code / OpenCode, and others) call `infer endpoint delete` through a skill, they run in a non-interactive environment:

- **hard-refuse by default**, even if `--yes` is provided.
- To delete in an agent / CI environment, the env must be set explicitly:

  ```bash
  ARKCLI_ALLOW_HEADLESS_DELETE=1 arkcli infer endpoint delete <id> --yes
  ```

- In an interactive human terminal, run `arkcli infer endpoint delete <id>` directly and confirm as prompted.

`--dry-run` emits the local `DeleteEndpoint` request plan. It calls neither
`GetEndpoint` nor `DeleteEndpoint` and does not enter the confirmation gate.
The plan does not prove that the endpoint exists, is deletable, or will be
accepted by the server.

### Recommendation before deletion

If the endpoint is in the `Running` state, some scenarios require `stop` before `delete`. When the status does not allow deletion, the backend returns `OperationDenied.EndpointStatus`, and arkcli translates it into "endpoint status does not allow this operation" and suggests using `infer endpoint get <id>` to check the status.

## Global Flags

| Flag | Type | Description |
|------|------|-------------|
| `--api-key` | string | ARK API key override |
| `--base-url` | string | Custom API base URL |
| `--debug` | | Print request and response debug details to stderr |
| `--format` | string | Output format: json (default "json") |
| `--page-all` | | Automatically fetch all pages when supported |
| `--page-delay` | int | Delay in milliseconds between pages (default 200) |
| `--page-limit` | int | Maximum pages to fetch with --page-all (default 10) |
| `--profile` | string | Active config profile |
| `--transform` | string | Transform output with a GJSON-style path expression |
