---
name: arkcli-gen
version: 2.0.0
description: "Image/video generation entry point: Use arkcli +gen when the user wants to generate an image, draw an image, generate an image/video, generate an image or a video from a reference image, generate from a reference image/video/audio, or explicitly use Seedream/Seedance models to create new content.+gen generates based on the available resources of the current profile and the model supported_params. Images are returned synchronously. After a video is submitted, task_id/status is returned. Use --wait or arkcli gen get/list to poll and download the result."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli +gen --help"
---

# arkcli generation workflow (+gen)

**CRITICAL: Before starting, you MUST first use the read tool to read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) (authentication gate, model lookup fallback, and shared security rules).**

**CRITICAL: This is a three-step workflow, not a single command. To generate images/videos, you MUST execute in the order `Step 1 → Step 2 → Step 3`. Do not skip Step 1/2 and run `+gen` directly: it will fail because the model name format is wrong (404) or because parameters unsupported by the model are passed. Before running it, be sure to read [`references/arkcli-gen.md`](references/arkcli-gen.md).**

## Why this is a workflow (core concept, understand before running)

When the user says "generate a video/image", it is essentially **three independent things that must be done in order**:

```
1. Which models are available in the current profile: Model sources differ completely by profile (EP vs model name)
2. Which parameters this model supports: Passing parameters without checking = guessing blindly = rejected by validation/backend
3: Generate using the available parameters
```

Compressing these three steps into "directly guess a `+gen` command" is exactly where failures come from: an incorrect model name format causes a 404, and unsupported parameters are rejected.

## Authoritative modality contract

Production calls resolve generation capability in this fixed order:

```text
explicit --modality > output_modalities > task types > unknown
```

- For a versioned model ID, Ark CLI reads `output_modalities` from ArkModels, then falls back to `task_types` / `filter_task_types` from the FoundationModel.
- For an `ep-*` ID, Ark CLI first reads `Endpoint.ModelReference.FoundationModel(name, version)`, then selects that exact model version and applies the same metadata contract.
- **Model names and display names are identifiers, not capability evidence.** Dola, Dreamina, Seedream, Seedance, or any future branding must not determine image/video routing.
- Missing or conflicting metadata resolves to `unknown`; pass `--modality image|video` explicitly instead of guessing.
- `+gen --dry-run` is a local Client Preview. It never reads live model or
  Endpoint metadata, calls generation APIs, downloads files, or opens an
  application. Pass `--modality image|video` for an unknown model or Endpoint;
  online-only routing is reported as `unresolved` with `fidelity=partial`.

## Use cases

- "Generate an image" / "text-to-image" / "draw an X".
- "Generate a video" / "text-to-video".
- Image-to-image / image-edit / adding a reference image; image-to-video (I2V); reference-to-video (R2V); reference audio.
- "Use this image as the first frame to generate a video" / "keep the motion from this reference video".

## Workflow overview

```
User intent: "Generate X"
  │
  ▼ Step 1 (mandatory) List models available in the current profile  ──► arkcli resources list --modality image|video
  │     platform    → List EPs (ep-xxx)           ┐
  │                                            ├─ Select one and record it as $MODEL
  │     coding-plan → List EPs (via platform)     ┘
  │
  ▼ Step 2 (mandatory, except EPs) Check available parameters for $MODEL  ──► arkcli models get $MODEL --transform supported_params
  │     Model name + has sp → **only** use the listed parameters, with values within min/max/enum
  │     Model name + empty sp (not configured or currently unparseable) → +gen automatically applies modality fallback defaults (video 720p/5s, image 2048)
  │     EP (ep-xxx)            → Skip; do not force-fill parameters (unknown underlying capabilities); let the server decide.
  │
  ▼ Step 3 Generate based on available parameters  ──► arkcli +gen --model $MODEL [parameters allowed by Step 2] "prompt"
  │
  ▼ Step 4 (result handling)
        Video = asynchronous: returns task_id + status=queued (**not a failure!**) → Use arkcli gen get <task_id> to poll; when it becomes succeeded, the result is automatically downloaded locally (local_path); add --wait for synchronous blocking
        Image = synchronous: directly returns output_url + local_path
```

## Step 1(mandatory) List models available in the current profile

```bash
# List by target modality. The output items[].id is the candidate that can be used as --model.
arkcli resources list --modality video   # or image
```

- **Platform differences (`resources list` already routes automatically by profile; just read items)**:
  - `platform` profile → items are **inference endpoints (EPs)** (`ep-xxx`), and each EP binds a model internally.
  - `coding-plan` profile → It does not contain vision models itself. `+gen image/video` automatically uses the platform data plane, and **`--model` must explicitly pass an EP on platform**.
- `is_default: true` marks the current default for this modality. Prefer it when the user does not specify one.
- **Select an ID and record it as `$MODEL`; use it throughout Step 2/3.**
- If the user has already explicitly provided a model/EP, it is still recommended to use `resources list` to check that it is available in the current profile. If it differs from the default, handle it according to "Default drift detection and promote nudge" in [`../arkcli-shared/references/profile-defaults.md`](../arkcli-shared/references/profile-defaults.md).

## Step 2 (mandatory, except EPs) Check available parameters for $MODEL

```bash
arkcli models get "$MODEL" --transform supported_params
```

- **`$MODEL` is a model name**: Get the `supported_params` list for this model (each item contains `name / type / support / min / max / enum / required`).
  - > **MUST: Step 3 can only use parameters where `support=true` here, and values must be within the `min/max/enum` range.** Parameters not in the list (or with `support=false`) will be rejected by `+gen`.
  - **You can directly use the ID returned by Step 1 `resources list`** (dot/display forms such as `dreamina-seedance-2-0` are all fine): `models get` automatically normalizes by DisplayName to the canonical hyphenated name. No manual conversion is needed. Only in very rare cases where `not found` is still reported should you use `arkcli models search <family name>` to check the name.
  - If the model is found but `supported_params` is empty / `null`, that version has no configured catalog or its upstream catalog is currently unparseable. Preserve any `warn: model supported_params enrichment failed: ...` line from stderr for troubleshooting. **Do not guess parameters manually**: `+gen` automatically uses built-in modality fallback defaults (video: `resolution=720p` / `duration=5` / `ratio=adaptive`; image: `size=2048x2048`) to fill parameters you did not specify. Go directly to Step 3.
- **`$MODEL` is an EP (`ep-xxx`)**: Skip this step. It is normal that supported_params cannot be found for an EP. Also, `+gen` **does not** apply fallback defaults to EPs (the model behind the EP may support higher capabilities, and forcing defaults may downgrade it incorrectly). Degrade open directly and let the server decide.
  - Only the `supported_params` lookup is skipped. A real `+gen` call still resolves image/video through `Endpoint -> ModelReference -> FoundationModel metadata`.
  - Provide `--modality` only when authoritative metadata is unavailable or conflicting.

## Step 3: Generate based on available parameters

```bash
# Text-to-image / text-to-video.
arkcli +gen --model "$MODEL" "<prompt>"

# With parameters confirmed in Step 2 (example: video 1080p + priority 9, provided that supported_params lists them).
arkcli +gen --model "$MODEL" --resolution 1080p --priority 9 "<prompt>"

# Image-to-image / image-to-video / reference assets: --input can be repeated.
arkcli +gen --model "$MODEL" --input @ref.jpg "<prompt>"
```

- For the full parameter set, multimodal `--input` rules, and the new `--n/--priority/--wait`, see [`references/arkcli-gen.md`](references/arkcli-gen.md).
- **Artifacts are automatically downloaded to CWD by default** (or `--save-to <dir>`). The `local_path` in JSON is the persistent artifact. The presigned `output_url` expires after 24 hours, so prefer referencing `local_path`.`--save-to=""` disables this.
- **Automatically open artifacts with the system default application**: By default, artifacts are opened only when stdout is an interactive terminal (a person runs it directly in the terminal). When an agent / pipeline / CI captures stdout (non-TTY), **no window pops up**, and only `local_path` is returned.`--open` forces opening, and `--no-open` forces not opening. This only applies to local files that have already been saved (when an asynchronous video does not use `--wait`, there is no local file and nothing is opened). For multiple artifacts, only the first few are opened.
- **🔑 You are an agent, so use `--open` by default**: When you (the AI agent) call arkcli, stdout is taken over by you = non-TTY, so the default auto behavior will not pop up a window. The user can only see the file path, not the finished artifact.**To let the user directly see the generated image/video, by default add `--open` to all `+gen` commands that generate images/videos for a real person and to `gen get` commands that poll to `succeeded`** (`--open` ignores TTY and forces opening on the user's desktop). Exceptions only apply when the user explicitly says "do not open it / in a script / batch / no pop-up", or when generating more than 4 images in one batch → in these cases, omit `--open` or explicitly use `--no-open`.

## Step 4 (result handling) Asynchronous video generation / synchronous image generation

| Modality | Default behavior | How you should read the result |
|------|---------|---------------|
| **Video** | **Asynchronous**: immediately returns `task_id` + `status: queued` | `queued` **is not a failure**. Use `arkcli gen get <task_id> --open` to poll until `succeeded`—**this `gen get` call will also download the artifact locally and return `local_path`** (default CWD, `<task-id>.mp4`). `--open` makes the finished artifact pop up directly on the user's desktop (you are an agent, non-TTY, so without it there is only a path). You do not need to manually curl `output_url`; **do not** resubmit `+gen` just because you did not get the video (that creates a new task) |
| Video + `--wait` | Synchronous: blocks until completion before returning | `arkcli +gen ... --wait --open`, directly get `output_url` / `local_path` and pop up the finished artifact |
| **Image** | **Synchronous**: directly returns `output_url` + `local_path` | `arkcli +gen ... --open` makes the image pop up directly for the user to view |

> **⚠️ Behavior change (2.0)**: The default video task behavior has changed from "automatically wait for completion" to "return task_id immediately after submission". To use the old synchronous blocking behavior, explicitly add `--wait`.

## Quick decision

- If the user wants to generate an image/video in one go → Use this workflow (Step 1→2→3).
- If the user has not chosen a model yet → Step 1 `resources list` lists candidates; if the model family is uncertain → switch to [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md).
- Image-to-image / reference assets → Add `--input @<file>` in Step 3 (repeatable).
- If the user "does not see the video" after video generation → It is probably asynchronous `queued`. Use `arkcli gen get <task_id> --open` to poll. The call that reaches `succeeded` will automatically download the artifact locally (see the returned `local_path`) and pop up the finished artifact. Do not resubmit.
- **Add `--open` by default when generating images/videos for a real person** → You are an agent (non-TTY). Without it, the user can only see the path, not the finished artifact. Omit it or use `--no-open` only for "do not open / in a script / batch >4 images".

## Natural language trigger table for advanced flags

| How the user says it | Corresponding flag / command |
|---|---|
| "Open it directly after generation / open it for me to see / pop it up when it is ready" | `arkcli +gen --open` (force opening with the system default application; it is already opened automatically by default in an interactive terminal) |
| "Do not open automatically / no pop-up / I am running it in a script, do not open it" | `arkcli +gen --no-open` (force not opening) |
| "Preview / do not actually submit / only view parameters / dry run / trial run / take a look first" | Run `arkcli +gen ... --dry-run --format json`; inspect `steps`, `unresolved`, and `fidelity`, and do not present a partial plan as server validation |
| "Force execution / skip validation / I know it is unsupported but want to try it" | `arkcli +gen --force` |
| "Coherent multiple images / in order / consistent style / 4-panel comic / sequential images" | `arkcli +gen --sequential` |
| "My previous tasks / generation history / task list / task status" | `arkcli gen list` (list all asynchronous generation tasks) |
| "Is that task finished / check progress / check status" | `arkcli gen get <task_id>`

## Command overview

| Command | Role |
|------|------|
| `arkcli resources list --modality image\|video` | **Step 1**: Models/EPs available in the current profile |
| [`arkcli models get <model> --transform supported_params`](../arkcli-models/SKILL.md) | **Step 2**: Check available model parameters |
| [`arkcli +gen`](references/arkcli-gen.md) | **Step 3**: Generate based on available parameters |
| [`arkcli +gen --stream`](references/image-stream.md) | Streaming NDJSON output for image tasks |
| [`arkcli gen get <task-id>`](references/gen-meta.md) | **Step 4**: Poll/query asynchronous video tasks |
| [`arkcli gen list`](references/gen-meta.md) | List/filter asynchronous generation tasks |
| [`arkcli gen delete <task-id>`](references/gen-meta.md) | Delete asynchronous generation tasks |

## Common fallbacks

- Model name reports `not found` → `models get` already automatically normalizes dot/display forms. If it still reports this, the name is probably truly wrong. Use `arkcli models search <family name>` to check.
- Parameter rejected (`param_not_supported`) → Go back to Step 2 and check `supported_params`. Use only the listed parameters. If you really need to force it, you can add `+gen --force` to skip validation (the server still makes the final decision).
- **Content blocked by moderation** (`ContentRiskBlocked` / `*SensitiveContentDetected` / sensitive-content hit / copyright) → This is not a parameter issue, and `--force` cannot bypass it. Adjust the prompt / sensitive content in the input assets and retry. For structured blocking causes + fix guidance, switch to `arkcli doctor error <code>` in [`../arkcli-doctor/SKILL.md`](../arkcli-doctor/SKILL.md) (full coverage for all 5 video-generation blocking subtypes).
- Authentication error → Switch to [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).

## References

- [arkcli-shared](../arkcli-shared/SKILL.md) — Authentication and global parameters (must read).
- [arkcli-models](../arkcli-models/SKILL.md) — Detailed explanation of Step 2 model query / `supported_params`.
- [references/arkcli-gen.md](references/arkcli-gen.md) — Full `+gen` parameters + multimodal + asynchronous semantics.
