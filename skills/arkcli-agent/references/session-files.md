# Session and Files

## Env / Session

```bash
arkcli agent env create --name arkcli-<domain>-env-<timestamp> --config '{Type: cloud, Networking: {Type: unrestricted}}' --format json
arkcli agent session create --agent-id <agent-id> --environment-id <env-id> --title arkcli-<domain>-session-<timestamp> --format json
arkcli agent session get <session-id> --format json
```

Session operations follow OpenTOP; Session resources/events/threads traverse the data plane directly.

When creating a cloud environment online, an explicit `Networking: {Type: unrestricted}` must be specified; only `{Type: cloud}` will be validated by the backend as missing `config.networking`.

### Session TOS Resource

A Session can mount an existing TOS directory at creation time. The CLI appends a resource with `Type=tos`, `TosBucket=<bucket>`, and `TosKey=<prefix>/`.

Use `--tos-path` only when the user explicitly asks for a TOS mount and provides the location. If the user asks for a mount without a location, request the complete `tos://<bucket>/<prefix>/` value instead of guessing it.

```bash
arkcli agent session create \
  --agent-id agent-xxx \
  --environment-id env-xxx \
  --tos-path tos://my-bucket/analysis/ \
  --format json
```

The protocol may be omitted as `my-bucket/analysis/`. The value must identify a directory: the prefix must be non-empty and end in `/`. Values such as `tos://bucket/`, `bucket`, or `bucket/path` fail local validation. If `--resource` is also provided, the TOS resource is appended to that array.

`arkcli +new session` and `arkcli +iterate` support the same flag. It mounts an existing directory; it does not create a bucket, upload objects, or grant access. Confirm that the account has TOS access and permission to read the bucket and prefix.

### Environment Setup Script

An Environment initialization script is stored in `Config.SetupScript`. Pass the script directly or use `@` to read a local file:

```bash
arkcli agent env create \
  --name arkcli-analysis-env \
  --config '{Type: cloud, Networking: {Type: unrestricted}}' \
  --setup-script @./bootstrap.sh \
  --format json
```

`--setup-script` overrides `Config.SetupScript`. Complex configuration can still be supplied with `--config @./env.yaml` or `--file`. The CLI normalizes common lower-case and snake-case fields such as `setup_script` and `networking` to the OpenTOP request shape.

### Session Overrides

A Session may override an existing Agent or Environment for that Session only. Non-null fields such as Agent `System`, `Tools`, `McpServers`, `Skills`, and `Multiagent`, and Environment `Config`, use the service's replacement semantics. Supply complete arrays instead of expecting them to merge with the base Agent.

```bash
arkcli agent session create \
  --agent-id agent-xxx \
  --environment-id env-xxx \
  --agent-overrides '{Type: agent_with_overrides, Tools: [...]}' \
  --environment-overrides @./environment-overrides.yaml \
  --format json
```

`--agent-overrides` maps to `AgentWithOverrides`; `--environment-overrides` maps to `Environment`. If an override omits `Id`, the CLI fills it from `--agent-id` or `--environment-id`. These are one-of request variants, so the CLI removes the corresponding `AgentId`/`AgentVersion` or `EnvironmentId` from the final request. Do not use `--agent` and `--agent-overrides` together. `arkcli +new session` and `arkcli +iterate` accept the same override flags.

### Environment Status Filtering

Currently, `arkcli agent env list` **does not support the `--status` flag**. Do not generate the following command:

```bash
arkcli agent env list --status active
```

`ListEnvironments` does not support a `Status` filter field either; the `Environment` objects only have an optional `ArchiveTime`.

To filter by status, fetch the full list first, then filter locally using the AI Agent in the `arkcli` CLI:

```bash
arkcli agent env list --page-all --format json
```

- `ArchiveTime` is non-empty: `archived`
- `ArchiveTime` is empty or missing: `active`
- Although the frontend domain type reserves `updating`, no reliable field is available for this state; do not claim support for `updating`.
- Do not filter based on the default page; status filtering must be combined with `--page-all`, and be mindful of the global `--page-limit` affecting the results.

`--page-all` defaults to `Limit=100`, and the next page is fetched using `NextPage` in the response. `--page-limit` limits the number of pages to request, not the number of results.

## Files and Session Resources

- Upload local files to the Files API: `arkcli agent file upload --path ./data.csv --purpose user_data --wait-active`.
- Register with URLs/TOS: `arkcli agent file upload --url https://... --purpose user_data` or `--url tos://bucket/path/file --tos '{bucket: b, prefix: arkfiles/}'`.
- `agent file upload/delete` support command-local `--dry-run`. It emits a zero-network `preview.v1` plan and never uploads, polls, or deletes. Delete preview does not require `--yes`; real deletion still requires explicit confirmation and `--yes`.
- Preprocess for videos/special files can include `--preprocess-configs`; an expiration time can be specified with `--expire-at <unix-seconds>`.
- Wait for upload completion: `arkcli agent file wait <file-id>`.
- List existing files: `arkcli agent file list --purpose user_data --limit 20`. Use `arkcli --page-all agent file list` for full traversal, with `limit=100` and fetching `has_more + last_id -> after` for subsequent requests.
- When a user provides a local path and wants to mount it to an existing session, complete the following steps:

```bash
arkcli agent session resources add <session-id> --path ./data.csv --mount-path data.csv
```

This involves uploading the file, waiting for it to become active, and then adding the session resource.
The `session resources add --path` command supports passing related parameters: `--purpose`, `--tos`, `--preprocess-configs`, `--expire-at`, `--wait-timeout`, and `--wait-interval`.

Boundaries:

- `session resources add` currently only supports `type=file`, `file_id`, and `mount_path`, and does not support the more complex `github_repository`/`memory_store` parameters in PRD.
- The backend automatically prepends `mount_path` with `/mnt/session/uploads/`. The CLI will warn about this in stderr, but will not modify the user-provided path; for example, if the path is `reports/data.csv`, the final path becomes `/mnt/session/uploads/reports/data.csv`. Avoid providing `uploads/` or paths containing `..` to avoid overwriting or invalid paths.
- `resources get` is a derived CLI capability: list resources and filter by `resource_id`, `file_id`, or `mount_path`.
- Currently, the data plane does not have native resource update/delete capabilities; the CLI remains unsupported.
- After `session resources add`, the backend copies the source file to the session-managed uploads path. Therefore, the `file_id` seen in `resources list` may differ from the source `file_id` passed during the addition.
- Do not use ArkBFF/NodeBFF to fetch files; the CLI only connects directly to `/api/v3/files` for the data plane.

## Example

```bash
arkcli agent file upload --path ./sales.csv --purpose user_data --wait-active --format json
arkcli agent session resources add sess-xxx --path ./sales.csv --mount-path sales.csv --format json
```
