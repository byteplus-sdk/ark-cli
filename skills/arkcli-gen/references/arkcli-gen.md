# +gen

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to learn about authentication, global parameters, and security rules.

Execution-layer documentation for image/video generation (`+gen` full parameters).**This is step 3 of the three-step workflow**. For the complete workflow (① `resources list` to check profile models → ② `models get` to check supported_params → ③ `+gen` to generate), see [`../SKILL.md`](../SKILL.md).

> **⚠️ Video tasks are asynchronous by default**: submission immediately returns a `task_id` (`status: queued`). Use `arkcli gen get <task_id>` to poll; add `--wait` to block synchronously. Image tasks return synchronously.

## Commands

> **⚠️ `--model` must be in the complete `<name>-<primary_version>` form (such as `seedream-5-0-260128` or `dreamina-seedance-2-0-260128`) or an endpoint ID (`ep-xxx`). Passing only the family name will result in a 404 `InvalidEndpointOrModel.NotFound`.**
>
> The `primary_version` format is not fixed. It is often a 6-digit date, but it may also be an 8-digit date, include a qualified prefix, be a short number, or even be an empty string (in which case the full ID is the family name itself). About half of models are not 6-digit versions. For details, see link 0 in [`../../arkcli-models/SKILL.md`](../../arkcli-models/SKILL.md).**Do not use your own regular expression to decide "whether it looks like a complete ID".**
>
> **Complete before calling (required):** If you are not sure whether `--model` is already in complete form, run this first:
>
> ```bash
> VER=$(arkcli models get <name> --transform 'primary_version' | tr -d '"')
> # The --transform output includes JSON double quotes, so you must strip them with tr -d. Otherwise, $VER will be "260128", causing an incorrect concatenation.
> MODEL=${VER:+<name>-$VER}; MODEL=${MODEL:-<name>}   # Falls back to <name> when VER is an empty string.
> arkcli +gen --model "$MODEL" "<prompt>"
> ```
>
> If you have just run `models search/list`, you can directly reuse the returned `primary_version` parameter without calling `get` again.

```bash
# 1) Text-to-image (T2I) — Seedream model
arkcli +gen --model seedream-5-0-260128 "Minimalist business laptop, dark blue, white background"

# 2) Image-to-image / image-edit (I2I) — use --input to mention a reference image
arkcli +gen --model seedream-5-0-260128 \
  --input @ref.jpg \
  "Keep the composition and change the background to a beach at dusk"

# Multiple reference images (pass --input multiple times)
arkcli +gen --model seedream-5-0-260128 \
  --input @style1.jpg --input @style2.jpg \
  "Blend the styles of the two images, with a Shiba Inu as the subject"

# Generate multiple candidates at once (image-count > 1 automatically enables sequential)
arkcli +gen --model seedream-5-0-260128 --image-count 4 "Four style variants of a futuristic city skyline"

# 3) Text-to-video (T2V)
arkcli +gen --model dreamina-seedance-2-0-260128 "A Shiba Inu running under cherry blossom trees, slow motion"

# 4) Image-to-video / first-frame-to-video (I2V) — the first image is used as the first frame by default
arkcli +gen --model dreamina-seedance-2-0-260128 \
  --input @first.jpg \
  "The camera slowly pulls back while the subject stays still"

# Explicitly specify first / last frames
arkcli +gen --model dreamina-seedance-2-0-260128 \
  --input first:@start.jpg --input last:@end.jpg \
  "Generate tween animation between these two frames"

# 5) Reference video (R2V) — use a video as a reference asset
arkcli +gen --model dreamina-seedance-2-0-260128 \
  --input ref:@reference.mp4 \
  "Keep the motion path of the reference video and replace the main character with a robot"

# 6) Reference audio — synchronize the video rhythm with the audio
arkcli +gen --model dreamina-seedance-2-0-260128 \
  --input ref:@beat.mp3 \
  "A montage of city night scenes that cuts to the beat"

# Advanced flags — image tasks
arkcli +gen --model seedream-5-0-260128 \
  --guidance-scale 7.5 --optimize-prompt --output-format png \
  "Product photography style"

# Advanced flags — video tasks
arkcli +gen --model dreamina-seedance-2-0-260128 \
  --frames 96 --camera-fixed --return-last-frame --draft \
  "Aerial shot of a city at night"

# Output full JSON
arkcli +gen --model dreamina-seedance-2-0-260128 --format json "Product advertising video"

# Custom inference endpoint: a real call resolves the bound model metadata
arkcli +gen --model ep-20260416234150-zsd4v "Minimalist business laptop"

# Add an explicit modality only for missing/conflicting metadata
arkcli +gen --model ep-20260416234150-zsd4v --modality video "A Shiba Inu is running"
```

## Parameter

| Parameter | Required | Type |Description|
|------|------|------|------|
| `<prompt>` |Yes| positional | Generation prompt (positional parameter, placed at the end of the command) |
| `--model` | No| string | **Complete versioned model ID** (such as `seedream-5-0-260128` or `dreamina-seedance-2-0-260128`) or inference endpoint ID (`ep-xxx`). Passing only the family name will directly result in a 404 `InvalidEndpointOrModel.NotFound`.**Can be omitted in 0.1.16+**: when omitted, it falls back to the active profile's `Resources.<modality>.Default` based on `--modality`. If it is not set, a hint is reported to guide you to run `arkcli profile set-default --modality <m> <id>`. |
| `--modality` | See description | string | Generation modality: `image` or `video`. Real calls use `explicit value > ArkModels output_modalities > FoundationModel task types > unknown`. |
| `--input` | No| string (repeatable) | Reference asset references, added to content[] in the order they appear. Local file `@<path>` / remote `https://...` `tos://...` are all supported. Optional role prefixes — shorthand: `first:` `last:` `ref:` `none:` (`none` explicitly ignores the role; the shorthand does not pass role on the wire, so the server infers it by position). SDK explicit forms: `first_frame:` `last_frame:` `reference_image:` `reference_video:` `reference_audio:` (these are actually written to the wire `content[].role` parameter).**Image tasks**: folded into an image union. **Video tasks**: the first image is the first frame by default, other images are reference images, video → ref_video, audio → ref_audio. |
| `--name` | No| string | Task name override |
| `--version` | No| string | Model version override |
| `--size` | No| string | Image output size, such as `1920x1920`. The backend rejects it if the pixel count is too small. |
| `--image-count` | No| int | Number of images output by an image task. When `>1`, it is automatically converted to `sequential_image_generation=auto + max_images=N`. |
| `--n` | No| int | Alias of `--image-count` (image tasks). If both are passed, `--n` takes priority. |
| `--ratio` | No| string | Output aspect ratio override, such as `16:9` / `9:16` / `1:1` (video tasks). |
| `--resolution` | No| string | Output resolution, such as `480p`, `720p`, or `1080p` (video tasks). |
| `--duration` | No| int | Video duration (seconds) |
| `--frames` | No| int | Number of video frames (overrides duration on supported models). |
| `--seed` | No| int | Random seed. The same seed can reproduce results. |
| `--watermark` | No| bool | Whether to add a watermark. |
| `--generate-audio` | No| bool | Whether to generate audio synchronously (video tasks). |
| `--guidance-scale` | No| float | Classifier-free guidance scale for image tasks, such as `7.5`. |
| `--optimize-prompt` | No| bool | Image tasks: enable server-side prompt optimization. |
| `--output-format` | No| string | Image output format: `jpeg` or `png`. |
| `--response-format` | No| string | Image response format: `url` (default) or `b64_json` (directly returns the image as a Base64 string and does not store a URL). |
| `--prompt-thinking` | No| string | Image tasks: prompt optimization thinking mode `auto` / `enabled` / `disabled` (takes effect only when `--optimize-prompt=true`). |
| `--prompt-mode` | No| string | Image tasks: prompt optimization execution mode `standard` / `fast`. |
| `--sequential` | No| string | Image tasks: sequential image mode `auto` / `disabled`. When left empty, it keeps the automatic behavior of `--image-count > 1` → `auto`. Passing `disabled` explicitly forces single-image generation, even if `--image-count > 1`. |
| `--stream` | No| bool | Image tasks: streaming NDJSON output. For details, see [`image-stream.md`](image-stream.md). |
| `--camera-fixed` | No| bool | Video tasks: fix the virtual camera. |
| `--return-last-frame` | No| bool | Video tasks: return the last-frame URL (for continuation generation). |
| `--draft` | No| bool | Video task draft mode: faster / cheaper / lower quality. |
| `--priority` | No| int | Video task scheduling priority 0-9. A higher value means higher priority.**Constrained by supported_params** — check whether the model supports it and the range in step 2 first (tested: seedance-2.0 / 2.0-fast support `[0,9]`, while 1.5-pro does not). If the model does not support it, passing it will be rejected by validation. |
| `--service-tier` | No| string | Video tasks: service tier. |
| `--safety-id` | No| string | Video tasks: safety identifier passed in by the caller. |
| `--execution-expires-after` | No| int | Video task server-side TTL (seconds). |
| `--callback-url` | No| string | Video tasks: the server POSTs notifications to this URL for lifecycle events (created/running/succeeded/failed). |
| `--wait` | No| bool | **Video tasks**: block until the task is complete before returning.**Default is false** — submission immediately returns a `task_id` for asynchronous polling (default behavior since 2.0; older versions waited synchronously by default). |
| `--extra-body` | No| string (JSON object) | Video task forward-compatibility channel: pass a JSON object string. Its keys are merged into the top-level request body, so you can pass through newly added server-side parameters without upgrading arkcli. Example: `--extra-body '{"new_field":"value"}'` |
| `--tools` | No| string (repeatable) | Tool switch. Currently supports `web_search`. |
| `--save-to` | No| string | Local directory for saving generated artifacts. Default is `.` (current directory). Pass `--save-to=""` to explicitly disable auto-download. Download failures do not block the main process. |

## Return values

**Default output** (simplified):

```json
{
  "kind": "image",
  "model": "seedream-5-0-260128",
  "status": "succeeded",
  "output_url": "https://...",
  "output_urls": ["https://..."],
  "local_path": "/work/ark-gen.jpeg",
  "local_paths": ["/work/ark-gen.jpeg"]
}
```

- `output_url` / `output_urls`: TOS presigned URLs. They **expire after 24 hours**, so do not reference them long term.
- `local_path` / `local_paths`: local absolute paths written after auto-download is triggered by `--save-to`. They are **persistent artifacts**, so use them instead of URLs when possible. These two parameters do not exist when auto-download is disabled (`--save-to=""`).
- `task_id`: only available for video tasks.**Videos are asynchronous by default** — submission immediately returns this id + `status: queued`. Use `arkcli gen get <task_id>` (or `arkcli api arkruntime.get_content_generation_task --params '{"id":"<task_id>"}'`) to poll until `succeeded`, and then retrieve `output_url`. Use `+gen --wait` to block synchronously.

**`--format json`**: outputs the full task object (without `local_path`).

In addition to common parameters such as `id / status / output_url / ratio / resolution / duration / frames / generate_audio`, the full object of a video task also echoes extra parameters returned by the server (they appear as needed and are determined by the model and server):

| Parameter |Description|
|---|---|
| `usage` | `{prompt_tokens, completion_tokens, total_tokens}` token usage telemetry |
| `revised_prompt` | The prompt finally sent by the server to the model (after optimization / safety filtering). |
| `subdivisionlevel` | Task subdivision level (note that the JSON key is word-style without underscores). |
| `fileformat` | Output file format (also word-style without underscores). |
| `safety_identifier` | Echo of the safety identifier passed in by the caller. |
| `tools` | List of tool types actually used by the task, for example `["web_search"]`. |

These parameters do not appear when the server does not return them (omitempty), and this does not affect extraction of core parameters such as `output_url` / `local_path`.

## Common errors

| Error | Cause | Fix |
|------|------|---------|
| `model is required` | `--model` is not specified. | A model name must be specified. |
| `Error code: 404 - InvalidEndpointOrModel.NotFound` | `--model` is a model family name (such as `seedream-5-0` or `dreamina-seedance-2-0`), and no family-name alias is registered for that family. | Use `arkcli models get <name> --transform 'primary_version'` to get the version number, concatenate it into `<name>-<primary_version>`, and then pass it. |
| Missing prompt | Positional parameter not provided. | The prompt is a required positional parameter. |
| `image generation failed: InvalidParameter` | Parameter error such as too few pixels in the image size. This is common with sizes such as `1024x1024`. | Use `1920x1920` or larger. Add `--debug` to view the underlying error if needed. |
| The video task has no result for a long time. | Video tasks are **asynchronous** by default and return `task_id` + `status: queued` (**not a failure**). | Use `arkcli gen get <task_id>` to poll until `succeeded`. **Do not** submit `+gen` again (that creates a new task). Use `+gen --wait` if you want to wait synchronously. |
| `warn: auto-download failed: ...`(stderr) | Failed to download `output_url` (network / 403, etc.). | Does not block the main process. `output_url` is still returned, and the user can manually recover with `curl -o file.ext "<output_url>"` (note the 24h validity). |

## Notes

- The prompt is a positional parameter. Put it at the end of the command, and it is recommended to wrap it in quotes.
- Model name, model version, display name, Endpoint name, and `ep-*` ID are separate identifiers. Routing uses structured capability metadata, never a brand prefix.
- `--input` is the data-driven multimodal entry point: image extensions go through the image channel, video extensions go through the video channel, and audio extensions go through the audio channel. Other extensions are **silently discarded** in video tasks (content-generation has no input_file slot) and are not recognized as reference images in image tasks.
- The order of multiple `--input` values in video tasks is meaningful: the first image is used as the first frame by default. If you want to express "this image is a reference asset, not the first frame", use the `ref:` prefix.
- For image generation, it is recommended to start from `1920x1920`. Sizes such as `1024x1024` may currently be rejected directly by the backend.
- Image generation usually completes in a few seconds, while video generation may take tens of seconds to several minutes.
- **Presigned URLs expire after 24h**: `output_url` can only be downloaded with HTTP `GET` within a short time (note that it is not `HEAD`, because the signature is bound to the method). For persistent storage, rely on `local_path` or download in time.
- Default file name: when there is a `task_id` (video), the task_id is used; otherwise, `ark-gen` is used. The extension is inferred from Content-Type (`.jpeg` / `.mp4`, etc.). Duplicate names automatically get `-1`/`-2` suffixes.
- For low-level troubleshooting or if you want to control the submission + polling cadence yourself: use the raw API directly: `arkcli api arkruntime.create_content_generation_task --params '{"model":"<id>","content":[{"type":"text","text":"<prompt>"},{"type":"image_url","image_url":{"url":"https://..."}}]}'` to get `id`, and then poll with `arkruntime.get_content_generation_task --params '{"id":"<id>"}'`.

## References

- [arkcli-gen](../SKILL.md) -- gen skill overview
- [arkcli-shared](../../arkcli-shared/SKILL.md) — Authentication and global parameters
