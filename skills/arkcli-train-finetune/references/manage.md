# Retrieve and Manage a Fine-Tuning Job

Use this reference for details, observation, and lifecycle operations when the job ID is known. To select an artifact by metrics, export a custom model, or continue to deployment, read [`export-deploy.md`](export-deploy.md) instead.

## Command summary

| Command | When to use | Common parameters |
|---|---|---|
| `arkcli train finetune get <job-id>` | Canonical command for retrieving full details | `--transform` |
| `arkcli train finetune status <job-id>` | Retrieve status; semantically equivalent to the detail query | `--transform` |
| `arkcli train finetune watch <job-id>` | Continuously monitor progress and metric changes | `--interval`, `--timeout`, `--quiet`, `--rich` |
| `arkcli train finetune metrics <job-id>` | Retrieve metric curves | `--metric`, `--from-step`, `--to-step`, `--output` |
| `arkcli train finetune logs <job-id>` | Retrieve training logs | `--tail`, `--since`, `--search`, `--follow`, `--output` |
| `arkcli train finetune trajectory list/get <job-id>` | Retrieve rollout trajectories recorded for an RL job | `--limit`, `--step`, `--sample-id`, `--full`, `--output` |
| `arkcli train finetune update <job-id>` | Update the name or description | `--name`, `--description` |
| `arkcli train finetune pause/resume/stop/delete` | Perform lifecycle operations | Confirm before writing, deleting, or terminating; add `--yes` only when required |

## Retrieve the current state first

Before any write operation, run:

```bash
arkcli train finetune get <job-id>
```

Confirm that the job ID, current phase, and requested operation are compatible. Follow the current command's `--help` and backend error for phase restrictions; do not maintain a static phase matrix.

Use the same direct `get` when the user asks why one named job cannot be found, and preserve the authoritative API error. Do not infer validity from date-like or readable parts of an `mcj-*` ID, search for similar jobs with `list`, or switch profile/project/region. If the user explicitly limits scope, stop after explaining the target error.

## Details and process observation

Choose the smallest command that satisfies the request:

```bash
arkcli train finetune get <job-id>
arkcli train finetune watch <job-id>
arkcli train finetune metrics <job-id>
arkcli train finetune logs <job-id>
arkcli train finetune trajectory list <job-id>
```

Inspect the corresponding `--help` before execution.

- Use `get` for a one-time state query and `watch` to wait continuously for a terminal phase.
- Query available metric names before filtering, then use the exact returned name.
- When `metrics` uses both `--from-step` and `--to-step`, `to-step` must be strictly greater than `from-step`.
- Logs and metrics can be large. Prefer filters, tailing, or `--output`; do not load the complete result into context.
- For an RL job created with `--enable-trajectory`, use `trajectory list` for summaries and `trajectory get` for a specific rollout. This path does not depend on the fine-tuning SDK, but it requires an SSO-authenticated profile with access to the job's trajectory TLS topic. If access fails, report the authentication or TLS authorization error instead of describing trajectory inspection as unsupported on BytePlus.

### Save logs for one named job

When the user supplies both an MCJ ID and an output path, run this directly:

```bash
arkcli train finetune logs <mcj-id> --output <path>
```

This is a read-only, single-job query. Do not stop after viewing `logs --help`; do not run `train finetune list --page-all`, scan jobs with Python, or suggest switching profile/project. Preserve the original error and stop. Inspect the current `logs --help` only if the target command reports incompatible syntax, then retry the same target command.

`--follow` polls continuously for an active job. If the job is already `Completed`, `Failed`, or `Terminated`, the CLI emits the current log snapshot and exits automatically. It also exits when an active job later becomes terminal and an idle poll has no new lines. A terminal job may use `--follow --format json` to return one bounded JSON snapshot; unbounded follow on an active job does not support single-document structured output.

### Retrieve the complete trajectory for one named job

```bash
arkcli train finetune trajectory list <mcj-id> --full
```

The hierarchy must include `train finetune trajectory list`; never try `arkcli train trajectory` or wander through unrelated help trees. `--full` returns complete rollout content. Add `--output <path>` only when the user supplies a path or agrees because the result can be large. If the command reports that trajectory logging was disabled, no trajectory exists, or authentication/TLS access failed, preserve that error and stop. Do not inspect other jobs, profiles, or MCP configuration.

## Troubleshoot a failed job

1. Run `get` and report the current phase, failure reason, and CLI-provided hint.
2. Use `logs` with `--tail`, `--since`, or `--search` to isolate the relevant error; write large results to a file.
3. Query `metrics` only when the job has produced valid metric data, and use exact metric names returned by the CLI.
4. Compare the training and validation data, model version, customization type, and hyperparameters with the original creation configuration.
5. Do not resubmit until the failure cause has been identified or the user explicitly accepts the remaining risk.

## TLS configuration

Use the complete `tls config` hierarchy to retrieve or manage fine-tuning TLS configuration:

```bash
arkcli train finetune tls config get
arkcli train finetune tls config enable
arkcli train finetune tls config disable
arkcli train finetune tls config delete
```

No `tls update` operation is available. Do not omit `config` and write `arkcli train finetune tls get`.

- `get` is read-only and can run without confirmation.
- `enable` creates a potentially billable TLS resource. Explain the cost impact and obtain confirmation first.
- `disable` stops new data from being written but retains existing data. Obtain confirmation first.
- `delete` permanently removes existing trajectory, custom log, trace, and explainability data. Obtain separate confirmation; add `--yes` for non-interactive execution only after confirmation.

## Job metadata

Before updating the name or description, display the job ID, current value, and target value, then obtain confirmation:

```bash
arkcli train finetune update <job-id> --help
arkcli train finetune update <job-id> --name <name>
```

Send only the fields the user requested to change.

## Pause, resume, stop, and delete

Inspect each current interface first:

```bash
arkcli train finetune pause --help
arkcli train finetune resume --help
arkcli train finetune stop --help
arkcli train finetune delete --help
```

Rules:

- Use `pause` when the user expects to continue the job later.
- Use `resume` primarily to restore a `Paused` job, making it the reversible counterpart to `pause`. When the backend allows, it may also retry `Failed` or `Terminated`; still follow the current job phase and API result.
- `stop` terminates the job irreversibly. Explain the impact clearly and obtain confirmation.
- `delete` removes the job record. Display the target job ID, name, and current phase, then obtain separate confirmation. **Enforce the phase client-side:** only terminal phases (`Completed`, `Failed`, or `Terminated`) can be deleted. For a non-terminal phase, refuse locally and instruct the user to run `train finetune stop` first, avoiding a backend `phase_mismatch` after the user confirms.
- If the agent environment requires `--yes`, add it only after user confirmation.
- If the phase does not allow an operation, follow the CLI or backend hint. Do not bypass the restriction through repeated retries.

## Training artifacts

If the user only wants to view artifacts, run:

```bash
arkcli train finetune artifacts list <job-id>
```

If the user wants to select the best artifact, export it, or deploy it, stop this flow and read [`export-deploy.md`](export-deploy.md).
