# Self-hosted environments and work queues

Use the implemented API contract, not proposed Worker Credential, lease_id, complete, or Worker-list APIs from the PRD.

## Create and bind

```bash
arkcli agent env create \
  --name enterprise-worker-env \
  --runtime-type self_hosted \
  --dry-run --format json
```

The create-only `--runtime-type` flag sets `Config.Type` and overrides the same field from `--config`/`--file`. The existing `--config '{Type: self_hosted}'` remains valid. Do not include cloud-only Networking, Packages, Env, or SetupScript. Explicit fields are not silently removed: compatibility is validated by the server. Do not treat `env update` as a cloud/self_hosted type-switch workflow.

Confirm the final create parameters, then capture the returned Environment ID and read it back with `env get`. Bind a Session using `agent session create --agent-id <agent-id> --environment-id <env-id>`. Creating an environment does not start a Worker, and creating a Session does not prove execution: an enterprise Worker must consume work and return tool results.

## Observe

```bash
arkcli agent env work stats <env-id> --format json
arkcli agent env work list <env-id> --limit 100 --page-all --format json
arkcli agent env work get <env-id> <work-id> --format json
```

- Calls use the `/api/v3/environments/{environment_id}/work` data plane with the current product/profile API Key and scope, not ArkBFF. Pair an explicit base URL with an explicit API Key. Keep credentials out of logs, source files, and reports.
- List returns `data` and optional `next_page`. Pass the opaque cursor via `--page`; it is not a numeric page. `--page-all` defaults to 100 items per page and at most 10 pages; use `--page-limit` and `--page-delay` as needed. A remaining `next_page` means the result is incomplete.
- No state/session_id/worker_id filter flags are implemented. Match a Session using each work item's `data.id`. In the current backend, Work ID equals Session ID; prefer actual returned IDs.
- States are `queued -> starting -> active -> stopping -> stopped`. Work `stopped` is not Session success; inspect Session/events or `+debug` for the business outcome.
- `depth` counts unclaimed queued work; `pending` counts polled but unacknowledged queued work, not running work. `workers_polling` counts named Workers polling during the last 30 seconds, not all Workers processing active tasks. Zero alone does not prove all Workers are offline. `oldest_queued_at` covers depth and pending.
- Get/list/stats are read-only and do not support `--dry-run`. Report API errors without switching accounts or products.

## Stop

Read the work item first. Preview without mutating:

```bash
arkcli agent env work stop <env-id> <work-id> --dry-run --format json
```

After the user confirms the target and impact, execute `agent env work stop <env-id> <work-id> --yes`. The default is graceful (`force=false`); add `--force` only if explicitly requested. Without `--yes`, real execution returns `requires_confirmation` without a request. Preview is zero-network, does not require `--yes`, and never substitutes for execution authorization or server validation.

The current stop contract accepts only `force`, not `mode/reason`. The command returns the server Work state without waiting for remote processes: graceful stop normally enters stopping; force marks stopped immediately but does not prove the enterprise Sandbox has exited or that late tool results are rejected. Read work again and inspect Worker logs to confirm cleanup. Do not automatically escalate to force.

## Boundary

ArkCLI manages environments/Sessions and observes/stops work; it does not run a resident Worker or execute remote bash/file tool calls locally. Use the official SDK Worker integration supplied by the console, with product-appropriate endpoints. Current integration uses API Keys; do not invent Environment Key commands or substitute Vault credentials. Worker deployment requires a user-selected isolated environment and explicit credential, permission, and working-directory boundaries.
