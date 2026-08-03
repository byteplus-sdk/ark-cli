---
name: gen-meta
description: Reference for the three standard CRUD commands arkcli gen get / list / delete, used with the +gen workflow to manage submitted asynchronous generation tasks (video / 3D / future extensions).
---

# gen get / list / delete

This is the "reconciliation side" of `+gen`: `+gen` submits generation tasks, while these three commands query task status afterward, list historical tasks, and delete completed tasks.

## When to use

- `gen get <task-id>`: You already have a `task_id` and want to retrieve the latest task status, including `output_url` / `usage` / `error`.**Once a task is `succeeded`, `gen get` automatically downloads the output to the local machine** (current directory by default, file name `<task-id>.mp4`), consistent with the persistence behavior of `+gen --wait`; `--save-to=""` can be disabled.
- `gen list`: List asynchronous tasks under the current account, with filtering by `status` / `model` / `service-tier` / `task-id`.
- `gen delete <task-id>`: Delete a completed task record. **Destructive and irreversible; must confirm with `--yes` when non-interactive**.

## Modality coverage

The endpoints behind these three commands (SDK `arkruntime.ListContentGenerationTasks` / `GetContentGenerationTask` / `DeleteContentGenerationTask`) are **modality-agnostic** on the server side:

- Video-generation tasks created by `+gen` through a model or Endpoint.
- Tasks created by 3D tasks (`Hyper3D-*`, `Hitem3D-*` model families).
- Any asynchronous generation task submitted through `arkcli api arkruntime.create_content_generation_task`.

However, **image tasks are not included** because they use the synchronous endpoint `arkruntime.generate_images`, which has no concept of a task list. This sync/async boundary comes from structured capability metadata, never from a model brand prefix.

## Command quick reference

```bash
# 1. Submit a video task with +gen (asynchronous by default, immediately returns task_id; add --wait for synchronous blocking).
arkcli +gen "A cyberpunk city at sunset" --model dreamina-seedance-... --duration 5

# 2. After getting task_id, query by ID separately (no polling, one-time fetch).
#    When the task has succeeded, the output is automatically downloaded to the current directory (<task-id>.mp4).
arkcli gen get tsk_xxxxx

# 2b. Save to a specified directory / disable automatic download.
arkcli gen get tsk_xxxxx --save-to ./out
arkcli gen get tsk_xxxxx --save-to ""   # Only check status; do not download.

# 3. List recent succeeded tasks (10 items on the first page).
arkcli gen list --status succeeded --page-size 10

# 4. Paginate.
arkcli gen list --status succeeded --page-size 10 --page-num 2

# 5. Filter by exact model match.
arkcli gen list --model dreamina-seedance-2-0-260128

# 6. Batch query multiple task IDs.
arkcli gen list --task-id tsk_a --task-id tsk_b

# 7. Delete.
arkcli gen delete tsk_xxxxx --yes
```

## Flag descriptions

### gen get

`<task-id>` is a positional argument. The flags have the same meaning as the download/open switches of `+gen`:

| flag |Description|
|---|---|
| `--save-to <dir>` | The output is saved to this directory when the task is `succeeded` (current directory `.` by default); `--save-to=""` disables automatic download. |
| `--open` / `--no-open` | Whether to open the downloaded output with the system default program. Default is auto: open only in an interactive terminal, and stay silent under agent / pipe / CI. |

> **🔑 agent uses `--open` by default**: when you (the AI agent) call `gen get`, stdout is not a TTY, so the default auto mode does not pop up a window. The user can only see `local_path`, not the final output. When polling a video for a human user, add `--open` **by default** to the `gen get` call that reaches `succeeded`, so the final output opens directly on the desktop (`--open` ignores TTY). Omit it or use `--no-open` only in scenarios such as "do not open / in scripts / batch operations."

Download behavior details:

- The file name uses `<task-id>` (if there is no extension, it is inferred from Content-Type; videos are usually `.mp4`).
- If a file with the same name already exists, **append `-1` / `-2` … to distinguish it**, with no overwrite or skip (repeating `gen get` on the same completed task keeps multiple copies, as expected).
- Download only when `output_url` is non-empty (that is, the task is `succeeded`); `running` / `queued` / `failed` is a no-op, with no file written to disk and no `local_path` returned.
- Download failures only print a warning to stderr and do not affect the status output of `gen get` (the output URL remains in `output_url`, and you can retry manually).

### gen list

| flag |Description|
|---|---|
| `--page-num <int>` | 1-indexed page number; if not provided, the server default is used. |
| `--page-size <int>` | Page size; if not provided, the server default is used. |
| `--status <enum>` | Filter task status: `succeeded` / `failed` / `running` / `queued` / `cancelled`. |
| `--model <id>` | Filter by model name or endpoint ID (exact match, not a prefix). |
| `--service-tier <tier>` | Filter by service tier setting. |
| `--task-id <id>` | Filter specific task IDs; can be passed multiple times. |

### gen delete

`<task-id>` is a positional argument.

| flag | Description |
|---|---|
| `--dry-run` | Emit a local `preview.v1` delete plan without contacting the backend or deleting the task. Always run this preview first. |
| `--yes` | Skip the interactive confirmation for the real delete. Use it only after the user gives a separate explicit confirmation for the exact task ID in the current turn. Never infer authorization from the earlier preview request. |

Safe Agent sequence:

1. Run `arkcli gen delete <task-id> --dry-run --format json` and explain the exact task that would be deleted.
2. Wait for a separate explicit confirmation from the user.
3. Only then run `arkcli gen delete <task-id> --yes --format json`.

Do not add `--yes` to the preview command, do not treat a request to preview as delete authorization, and do not continue to the real delete without the second confirmation.

## Output format

- `gen get` reuses the `VideoTask` structure of `+gen`: `{id, model, status, output_url, ratio, resolution, duration, ...}`; when the task is `succeeded` and automatic download succeeds, `local_path` is also returned (the absolute path on disk; this parameter is omitted when `--save-to=""` is used or the task has not succeeded).
- `gen list` returns `{total: <int>, items: [<item>, ...]}`; each `item` is the raw JSON form of an SDK list item. **Note the following 3 differences from `gen get`**:
  - The error parameter is called `failure_reason`, not `error`.
  - **Does not include `resolution` / `ratio` / `duration`** (to get these parameters, call `gen get` again).
  - The JSON keys for the two parameters `subdivisionlevel` / `fileformat` are word-style (no underscores), reflecting the SDK wire truth.
- `gen delete` returns `{id, deleted:true}`. The server has no response body; this is synthesized by arkcli.

## Common errors

| Symptom | Cause |
|---|---|
| `gen get` / `gen delete` reports `missing task id`. | The positional arg was not provided. |
| `gen list` returns empty `items[]`. | The filter is too strict, or there are indeed no tasks under the current account. |
| `gen get` cannot get `output_url`. | The task is still `running` / `queued`; check the `status` parameter. |
| Using the ID returned by an image task with `gen get` reports not found. | Images use a synchronous endpoint, not a task list; the usage is mismatched. |

## Relationship with the raw API

| Subcommand | Equivalent raw API |
|---|---|
| `gen get <id>` | `arkcli api arkruntime.get_content_generation_task --params '{"id":"<id>"}'` |
| `gen list ...` | `arkcli api arkruntime.list_content_generation_tasks --params '{"page_num":1,"filter":{...}}'` |
| `gen delete <id>` | `arkcli api arkruntime.delete_content_generation_task --params '{"id":"<id>"}'` |

CLI subcommands are human-friendly wrappers around the raw API; the commands are shorter and the flags are more intuitive. The raw API is still the fallback channel.

## Use with `+gen`

- When `+gen` itself times out while polling, it returns `task_id` and a hint. Continue tracking with `gen get <task-id>`, which is more cost effective than rerunning `+gen` (rerunning creates another new task and consumes quota again). The `gen get` call that polls to `succeeded` downloads the output locally along the way, so you do not need to manually curl `output_url`.
- Asynchronous submission (without `--wait`) + polling with `gen get` is the mainstream video workflow: `arkcli +gen ... --modality video` gets `task_id` → repeatedly run `gen get <id>` until `succeeded` → the output is automatically saved locally. With `--wait`, it is the synchronous blocking version; the two paths have the same persistence behavior.
- `gen list` is the only entry point for checking "whether that task I submitted earlier actually exists"; if `+gen` does not get a normal return after submission, it is recommended to run `gen list --status running` first.
