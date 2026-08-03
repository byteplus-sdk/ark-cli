# +understand command reference

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) (authentication, API Key error recovery, global parameters, safety rules) and [`sub-skills.md`](sub-skills.md) (purpose and expected output of 12 sub-skills).

`+understand` is a "one engine + semantic layer" multimodal understanding workflow built on the data-plane Responses API (`POST /responses`). It shares the same `CreateResponses` engine with `+chat`; every sub-skill provides an expert system prompt and a plan-profile fallback model. A Platform Profile instead uses its deployed `Resources.Text.Default` endpoint when `--model` is omitted.

## Command form

```bash
arkcli +understand <sub-skill> --input @file [prompt]   # Explicit sub-skill
arkcli +understand "<prompt>" --input @file             # Omit sub-skill → automatically route by --input modality
```

```bash
# Image description (minimal)
arkcli +understand image-caption --input @photo.jpg "Describe the main subject of the image in Chinese"

# Visual localization (bbox output)
arkcli +understand image-grounding --input @scene.jpg "Draw bounding boxes around all pedestrians in the image"

# Document extraction (by fields)
arkcli +understand doc-extract --input @invoice.pdf "Extract: invoice number / amount / invoice date"

# Speech transcription (no prompt required; task instructions are built into the sub-skill)
arkcli +understand asr --input @speech.mp3

# Generate SRT subtitles
arkcli +understand asr-align --input @speech.mp3

# Chapter-based video summary
arkcli +understand video-summary --input @clip.mp4 "Organize the output by chapter and provide a timestamp for each chapter"

# Omit sub-skill: automatically route by first --input modality (here .mp3 → asr)
arkcli +understand "Transcribe this recording" --input @speech.mp3

# Incremental streaming output
arkcli +understand asr --input @speech.mp3 --stream

# Append instructions after the built-in prompt (does not replace the expert prompt)
arkcli +understand image-caption --input @x.jpg --system-prompt-append "Answer in one sentence, in English"

# Complete JSON output
arkcli +understand asr --input @speech.mp3 --format json
```

## Positional argument parsing (sub-skill vs prompt)

`+understand` requires at least one positional argument (`MinArgs=1`):

- If `args[0]` matches the registry (one of 12), it is the sub-skill; `args[1:]` are joined as the prompt (layered on the built-in prompt as additional user requirements).
- If `args[0]` does not match, all `args` are the prompt, and the server **automatically routes** by the modality of the first `--input` (image→image-caption / video→video-qa / audio→asr / file→doc-extract).
- Many sub-skills such as `asr` / `asr-align` are fully defined by built-in prompts, so prompt may be omitted. However, the **number of positional arguments must still be ≥1**, so at least the sub-skill name is required.

## Parameters

> Strictly matches `arkcli +understand --help`. The `+understand` flag set is **smaller than `+chat`**: it replaces `+chat`'s `--instructions` with `--system-prompt-override/append`; it has **no** `--tools` / `--tools-file` / `--tool-choice` / `--caching` / `--cache-prefix` / `--text-format` / `--text-schema` / `--previous-response-id` / `--include-events` / `--expire-at`. Do not copy these flags from `+chat`.

| Parameter | Required | Type | Description |
|------|------|------|------|
| `<sub-skill>` | Yes* | positional | One of 12 (see [`sub-skills.md`](sub-skills.md)). When omitted, route automatically using first `--input`, but total positional arguments must still be ≥1 |
| `[prompt]` | No | positional | User requirements added above the built-in system prompt, such as extraction fields, a specific question, output language |
| `--input` | Yes | string (repeatable) | Input such as `@photo.jpg`; extension routes to image/video/audio/file. **Without it, returns `missing_input`** |
| `--model` | No | string | Explicit resource override and highest priority. When omitted, a Platform Profile uses its `Resources.Text.Default` `ep-xxx`; a plan profile uses the sub-skill fallback model. An explicit value must be a full versioned ID or `ep-xxx` |
| `--stream` | No | bool | Streaming output in two sections: `Thinking:` + `Response:` |
| `--system-prompt-override` | No | string | **Completely replace** built-in system prompt |
| `--system-prompt-append` | No | string | **Append** extra instructions to built-in prompt |
| `--temperature` | No | float | Sampling temperature, such as `0.7` |
| `--top-p` | No | float | Nucleus sampling top_p, such as `0.9` |
| `--max-output-tokens` | No | int | Reply token limit |
| `--reasoning-effort` | No | string | Reasoning intensity: `minimal` / `low` / `medium` / `high` (only supported models) |
| `--thinking` | No | string | Internal reasoning: `auto` / `enabled` / `disabled` |
| `--store` | No | bool | Persist response (retrieve later with `arkcli chat get <id>`) |
| `--no-progress` | No | bool | Disable stderr heartbeat for non-streaming calls (scripts) |
| `--timeout` | No | duration | Total limit for local-file preparation and inference. Default: `2m`. Pass a longer value such as `10m` explicitly for large files. |
| `--dry-run` | No | bool | Resolve the recipe, persisted Profile endpoint, explicit model, input references, and request summary locally. It does not read live metadata, upload files, call Responses, produce usage/response ID, or persist a response. |

\* At least a sub-skill name or routable `--input` is required; otherwise `missing_sub_skill`.

See [`../../arkcli-shared/references/global-flags.md`](../../arkcli-shared/references/global-flags.md) for global flags (`--profile` / `--api-key` / `--base-url` / `--format json` / `--transform` / `--debug`, and others).

## Model/endpoint resolution order

```text
explicit --model
  -> Platform Profile: active Profile.Resources.Text.Default (must be ep-xxx)
  -> plan profile: sub-skill recipe fallback model
```

- A Platform Profile without `Resources.Text.Default` fails with guidance to configure a default resource; it must not silently fall back to a bare model name.
- A missing default must not make the Agent switch Profiles, replace the sub-skill, or route to another business command.
- `--dry-run` and real execution use the same resolved resource, but preview reads only local Profile configuration and performs no live Endpoint discovery.

## System prompt resolution chain

Every sub-skill has a built-in system prompt injected into Responses `instructions`:

```
Built-in prompt (by sub-skill)
  → when --system-prompt-override is non-empty: replace entirely with override
  → when --system-prompt-append is non-empty: append to previous result (one blank line between)
```

- To **fine-tune** a task (language/output constraints): use `--system-prompt-append` and preserve the expert prompt.
- For **complete customization**: use `--system-prompt-override`, effectively reducing the sub-skill to `+chat` with a specified model + custom prompt.
- The `[prompt]` positional argument is a **user message**, separate from the system prompt: the sub-skill defines "who you are/how to work"; prompt defines "what this request needs".

## `@file` upload mechanism (audio is special)

`--input @<path>` infers modality by extension:

- `.jpg/.jpeg/.png/.webp/.gif/.bmp` → image
- `.mp4/.mov/.webm/.mkv/.avi` → video
- `.mp3/.wav/.m4a/.aac/.flac/.ogg` → audio
- Others → file (such as `.pdf`)

Supported references: `@<path>` (local, resolved to absolute `file://`), `<path>` (without prefix, equivalent to `@<path>`), `https://` / `http://` (remote URL), and constructed `file://` / `tos://` URLs.

Local inputs use product-safe preparation paths:

- **image / video / doc, aggregate size up to 45 MiB**: arkcli uses Responses-native inline forms (`image_url` / `video_url` data URLs and `input_file.file_data`). This avoids a separate Files API request on BytePlus plan endpoints while leaving headroom below the 64 MiB request-body limit after base64 expansion.
- **image / video / doc above 45 MiB**: arkcli uploads through Files API and waits for file status before inference. Only `active` files are attached to Responses as `file_id`. A `failed` file returns its file-service code/message and exits non-zero before inference; arkcli never sends that failed file to the model.
- **audio**: the SDK preprocessor **does not process audio**; arkcli inlines local audio as a **base64 data URL** before sending.

`--timeout` covers both file preparation and Responses inference for streaming and non-streaming execution. A timeout returns the stable `understand_timeout` error type and exits non-zero. Agents must not treat it as success or wait without a bound.

### Audio size constraints

| Form | Limit | Current state |
|---|---|---|
| base64 (`@local-audio`, inline data URL) | **25 MB** | Current implementation; **oversize is not blocked or warned** (product decision), so large files may fail or be rejected |
| Public URL (`https://...`) | 25 MB | The backend downloads it, so the URL must be reachable from the service environment |
| Files API (file_id, 512 MB) | 512 MB | Pending SDK upgrade; unavailable now |

> For audio **over 25 MB**, warn first and recommend splitting or reducing bitrate. Do not pretend large files will definitely work.

## Return value

> **Identical to `+chat`: arkcli flat schema, not native Responses `output[].content[].text`.** Read assistant text directly from `.content`. Global `--format` accepts only `json` and does not restore the native nested shape.

`--dry-run` is the exception: it returns `preview.v1` rather than a model response and never includes model-response `id`, `content`, or `usage`:

```json
{
  "dry_run": true,
  "mode": "local",
  "validation_level": "local",
  "validated": true,
  "plan": {
    "operation": "arkruntime.create_responses",
    "recipe": "image-caption",
    "model": "dola-seed-2.0-lite",
    "prompt": "Describe the image",
    "inputs": [{"kind": "image", "source": "file:///abs/photo.jpg"}],
    "stream": false,
    "instructions_source": "recipe"
  }
}
```

**Default/`--format json` output**:

```json
{
  "id": "resp_021776932247884ea9dc72e4279a1799608cb64438f5f0f125439",
  "model": "seed-2-0-lite-260428",
  "content": "...",
  "reasoning_content": "...",
  "usage": { "prompt_tokens": 88, "completion_tokens": 419, "total_tokens": 507 }
}
```

- `content`: sub-skill output (transcription / bbox JSON / SRT / extraction JSON / minutes Markdown, and others), **flattened to a string**. See [`sub-skills.md`](sub-skills.md).
- `reasoning_content`: reasoning process (non-empty only for reasoning models).
- `id`: response ID; after `--store`, retrieve with `arkcli chat get <id>`.

**Streaming output** (`--stream`):

```
Thinking:
<reasoning deltas...>

Response:
<response deltas...>
```

Streaming prints directly without JSON and ends with a newline.

### Parse with jq

```bash
# Extract main output (transcription / JSON / SRT ...)
arkcli +understand asr --input @speech.mp3 | jq -r .content

# JSON output such as bbox/extraction: content is a JSON string and requires a second parse
arkcli +understand image-grounding --input @x.jpg "Draw bounding boxes around everyone" | jq -r .content | jq .

# Extract token usage
arkcli +understand asr --input @speech.mp3 | jq .usage
```

Do not use `jq '.output[].content[].text'`; the output is flat. Model-generated JSON is a string in `.content`; first extract it with `jq -r .content`, then parse again with `jq .`.

## Common errors

| Error code / message | Cause | Handling |
|------|------|------|
| `missing_sub_skill`: `understand needs a sub-skill or an input file to auto-route` | No sub-skill and no routable `--input` | Add a sub-skill or `--input @file` |
| `unknown_sub_skill`: `unknown understand sub-skill "X"` | Misspelled/nonexistent sub-skill | Use one of 12 valid names in [`sub-skills.md`](sub-skills.md) |
| `missing_input`: `<sub> requires an input file via --input` | Sub-skill given without input | Add `--input @your-file` |
| `unroutable_input`: `cannot auto-route a "X" input` | No default mapping for inferred modality | Specify sub-skill explicitly |
| `input file not found: <path>` | Path does not exist or is directory | Check path/current directory |
| `understand_timeout` | File preparation or inference exceeded the client deadline | Validate the file and model modality first; pass `--timeout 10m` explicitly for a large file; retain `--debug` output for a repeatable failure |
| `uploaded responses file <id> failed preprocessing` | Files API marked the local input as `failed` | Use the returned code/message to inspect file content/format; never retry the same failed `file_id` |
| `ark runtime: API Key is required` | API Key not configured | `arkcli auth apikey` or set `ARK_API_KEY` |
| `Error code: 403 - {"code":"AccessDenied",...}` | Data-plane authentication/permission | **Do not retry**; follow API Key recovery in [`../../arkcli-auth/references/auth-modes.md`](../../arkcli-auth/references/auth-modes.md) |
| `Error code: 404 - InvalidEndpointOrModel.NotFound` | Unresolvable family name passed to `--model` | Use full `<name>-<primary_version>` or `ep-xxx`; see [`../../arkcli-models/SKILL.md`](../../arkcli-models/SKILL.md) |

## Direct `arkcli api` call (raw fallback)

The operation behind `+understand` is `+chat`'s `arkruntime.create_responses` / `arkruntime.create_responses_stream`. A sub-skill is a particular `model` + `instructions` and can be reproduced manually:

```bash
arkcli api arkruntime.create_responses --params '{
  "model": "seed-2-0-lite-260428",
  "instructions": "<Paste the built-in system prompt for the corresponding sub-skill here>",
  "input": [{
    "role": "user",
    "content": [
      {"type": "input_audio", "audio_url": "file:///abs/path/speech.mp3"},
      {"type": "input_text", "text": "<Optional additional requirements>"}
    ]
  }]
}'
```

In raw mode, arkcli no longer automatically inlines audio as base64. Construct a backend-accessible URL or data URL yourself. Prefer `+understand` whenever possible. See [`../../arkcli-api-explorer/SKILL.md`](../../arkcli-api-explorer/SKILL.md).

## Relation to other commands

- **[`arkcli +chat`](../../arkcli-chat/SKILL.md)**: Open conversation/reasoning, multi-turn continuation, tools, and `--text-format json_schema`. To follow up on `+understand`, use `--store`, then `+chat --previous-response-id <id>` (same engine).
- **[`arkcli +gen`](../../arkcli-gen/SKILL.md)**: Image/video **generation**, not understanding existing material.
- **[`arkcli models`](../../arkcli-models/SKILL.md)**: Query a full model version when overriding `--model`.

## Security and privacy

- `--input @<file>` uploads/inlines the **entire file** (Responses-native inline data for BytePlus image/video/doc up to 45 MiB, Files API above that threshold, and base64 for audio). Do not upload sensitive material casually; the service receives the complete content.
- `--dry-run` only checks the file locally and prints its reference summary; it does not upload or inline the file. The JSON may contain an absolute local `file://` path, so redact logs before sharing.
- `--debug` prints requests/responses (possibly base64 audio and file URLs) to stderr; redact logs before sharing.
- Responses persisted with `--store` remain server-side; delete when needed with `arkcli chat delete <id>`.
