# +chat

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to learn about authentication, global parameters, and security rules.

A high-level wrapper for the data-plane Responses API (`POST /responses`). A single command returns the assistant response and supports multimodal input, reasoning, streaming, and multi-turn conversations.

## Commands

> **⚠️ `--model` must be in the complete `<name>-<primary_version>` form, such as `dola-seed-2-1-turbo-260628` or `seed-2-0-lite-260428`, or an endpoint ID (`ep-xxx`). Passing only the family name, such as `dola-seed-2-1-turbo` or `glm-5-2`, will return 404 `InvalidEndpointOrModel.NotFound`.**
>
> The `primary_version` format is not fixed. It is commonly a 6-digit value such as `260628`, but it can also be a 6-digit value such as `260428`, a prefixed value such as `preview-260328`, a short number such as `2507`, or even an empty string (in that case, the full ID is the family name itself). About half of models are not 6-digit versions. For details, see link 0 in [`../../arkcli-models/SKILL.md`](../../arkcli-models/SKILL.md).**Do not use your own regular expression to decide "whether it looks like a complete ID".**
>
> **Complete before calling (required):** If you are not sure whether `--model` is already in complete form, run this first:
>
> ```bash
> VER=$(arkcli models get <name> --transform 'primary_version' | tr -d '"')
> # The --transform output includes JSON double quotes, so you must strip them with tr -d. Otherwise, $VER will be "260628", causing an incorrect concatenation.
> MODEL=${VER:+<name>-$VER}; MODEL=${MODEL:-<name>}   # Falls back to <name> when VER is an empty string.
> arkcli +chat --model "$MODEL" "<prompt>"
> ```
>
> If you have just run `models search/list`, you can directly reuse the returned `primary_version` parameter without calling `get` again.

```bash
# Plain-text chat
arkcli +chat --model dola-seed-2-1-turbo-260628 "Introduce yourself in three sentences"

# Streaming output (think first, then response).
arkcli +chat --model dola-seed-2-1-turbo-260628 --stream "Explain quantum entanglement"

# System-level instructions + sampling adjustment
arkcli +chat --model dola-seed-2-1-turbo-260628 \
  --instructions "You are an assistant that answers in only one sentence" \
  --temperature 0.2 --top-p 0.9 --max-output-tokens 128 \
  "Why is the sky blue?"

# Increase reasoning effort (only effective on models that support reasoning)
arkcli +chat --model dola-seed-2-1-turbo-260628 --reasoning-effort high \
  "Prove: there are infinitely many prime numbers"

# Multimodal input (local image)
arkcli +chat --model dola-seed-2-1-turbo-260628 --input @photo.jpg "Describe this image"

# Multimodal input (audio)
arkcli +chat --model dola-seed-2-1-turbo-260628 --input @clip.mp3 "What is this audio saying?"

# Multiple files (comparison and combined understanding)
arkcli +chat --model dola-seed-2-1-turbo-260628 --input @img1.jpg --input @img2.jpg "Compare these two images"

# Use an Endpoint ID (endpoint) instead of the model name
arkcli +chat --model ep-20260416234150-zsd4v "hello"

# Multi-turn continuation — you must use --store to persist the first turn first.
RESP_ID=$(arkcli +chat --model dola-seed-2-1-turbo-260628 --store --format json "First-turn question" \
  | jq -r .id)
arkcli +chat --model dola-seed-2-1-turbo-260628 --previous-response-id "$RESP_ID" "Second-turn question"

# Full JSON output
arkcli +chat --model dola-seed-2-1-turbo-260628 --format json "hello"
```

## Parameter

| Parameter | Required | Type |Description|
|------|------|------|------|
| `<prompt>` |Yes| positional | Prompt (positional parameter, placed at the end of the command) |
| `--model` | No| string | **Complete versioned model ID** (such as `dola-seed-2-1-turbo-260628`) or inference endpoint ID (such as `ep-xxx`). Passing only the family name `dola-seed-2-1-turbo` is unstable, and newer model families will directly report `InvalidEndpointOrModel.NotFound`.**0.1.16+ can be omitted**: If omitted, it falls back to `Resources.Text.Default` in the active profile, resolved in the order `--profile > ARK_PROFILE > default`. If it is not set, a hint guides you to run `arkcli profile set-default --modality text <id>` |
| `--input` | No| string (repeatable) | File reference, such as `@photo.jpg`; routed to an image/video/audio/file ContentItem by extension. Passing it repeatedly means multiple files. |
| `--stream` | No| bool | Streaming output (two sections: `Thinking:` + `Response:`) |
| `--instructions` | No| string | System-level instructions injected into this Responses request. |
| `--temperature` | No| float | Sampling temperature, such as `0.7`. |
| `--top-p` | No| float | Nucleus sampling top_p, such as `0.9`. |
| `--max-output-tokens` | No| int | Maximum number of tokens in the assistant response. |
| `--reasoning-effort` | No| string | Reasoning effort: `minimal` / `low` / `medium` / `high` (only effective on models that support reasoning). |
| `--store` | No| bool | Persist this response so that later `--previous-response-id` can continue from it. Without `--store`, only a very short window is retained. |
| `--previous-response-id` | No| string | Previous response ID, used for multi-turn conversation continuation. |

## @file mechanism

Processing flow for `--input @<path>`:

1. arkcli converts `@photo.jpg` to an **absolute path** `file:///Users/.../photo.jpg`.
2. Infer the mime type by extension:
   - `.jpg/.jpeg/.png/.webp/.gif/.bmp` → `input_image`
   - `.mp4/.mov/.webm/.mkv/.avi` → `input_video`
   - `.mp3/.wav/.m4a/.aac/.flac/.ogg` → `input_audio`
   - Other extensions → `input_file`
3. The underlying SDK automatically performs the "upload → get file_id → replace URL with file_id" flow, and then sends the request.
4. This is transparent to the user.

Supported reference forms:
- `@<path>` — Local file. The path is resolved to an absolute path.
- `<path>` (without a prefix) — Equivalent to `@<path>`.
- `https://...`, `http://...` — Remote URL, passed directly to the model.
- `file://...`, `tos://...` — Preconstructed URL.

## Return values

> **Important: The `+chat` output has been flattened by arkcli into scalar parameters. It is not the native Responses API nested structure `output[].content[].text`.** The assistant text is presented directly as the `content` string. The service layer concatenates all `output_text` fragments into a single section. Reasoning text likewise uses `reasoning_content`. The global `--format` only accepts `json` and **does not** switch back to the native nested shape. If you use `output[].content[].text`, `jq` will not get anything.

**Default output** (flat schema):

```json
{
  "id": "resp_021776932247884ea9dc72e4279a1799608cb64438f5f0f125439",
  "model": "dola-seed-2-1-turbo-260628",
  "content": "...",
  "reasoning_content": "...",
  "usage": {
    "prompt_tokens": 88,
    "completion_tokens": 419,
    "total_tokens": 507
  }
}
```

- `id`: The response ID of this request, used for the next `--previous-response-id`.
- `content`: The assistant's final answer text (**already flattened into a string**, with all `output_text` fragments concatenated by the service).
- `reasoning_content`: The model's reasoning process text (non-empty only when the model supports reasoning).
- `usage`: Token usage.

**Streaming output** (`--stream`):

```
Thinking:
<Incremental reasoning text...>

Response:
<Incremental final answer text...>
```

In stream mode, JSON is not output. The result is printed directly, followed by a newline when it ends.

### Parse with jq (common)

```bash
# Get the assistant text body
arkcli +chat --model <id> "<prompt>" | jq -r .content

# Get reasoning text (non-empty only for reasoning models)
arkcli +chat --model <id> --reasoning-effort high "<prompt>" | jq -r .reasoning_content

# Get token usage
arkcli +chat --model <id> "<prompt>" | jq .usage

# Get the response ID (used with --store to chain multiple turns)
arkcli +chat --model <id> --store "<prompt>" | jq -r .id
```

> Do not write `jq '.output[].content[].text'`. `+chat` has been flattened and does not have that path.

### Get the native SDK shape (fallback)

If you must parse the native Responses API nested `output[]` structure, bypass `+chat` and use the raw API explorer:

```bash
arkcli api arkruntime.create_responses --params '{
  "model": "<id>",
  "input": [
    {"role": "user", "content": [{"type": "input_text", "text": "Hello"}]}
  ]
}'
```

The returned value is the arkcli-side `CreateResponsesResponse` (registered as a raw operation), which preserves parameters closer to the upstream SDK, such as `output[]`, `usage`, and `reasoning`. For details, see [`../../arkcli-api-explorer/SKILL.md`](../../arkcli-api-explorer/SKILL.md).

## Common errors

| Error | Cause | Fix |
|------|------|---------|
| `prompt is required when no --input is provided` | A prompt is required for plain-text chat. | Add the prompt positional parameter. |
| `model is required` | `--model` is not provided. | Add a model name or endpoint ID. |
| `input file not found: <path>` | The file for `@<path>` does not exist or is a directory. | Check the path. When using a relative path, pay attention to the current working directory. |
| `ark runtime: API Key is required` | The API key is not configured.| Run `arkcli auth apikey` or set the `ARK_API_KEY` environment variable. |
| `Error code: 400 - ... invalid scheme` | The URL passed in is not recognized by the backend (for example, `file://` was not uploaded successfully by the SDK). | Check the file size and permissions. Add `--debug` to view the SDK upload flow. |
| `Error code: 400 - ... model not found` | The `--model` name or endpoint ID is incorrect. | Check with `arkcli models search` / `arkcli infer endpoint list`. |
| `Error code: 404 - InvalidEndpointOrModel.NotFound` | The value passed to `--model` is a model family name, such as `dola-seed-2-1-turbo` or `glm-5-2`, and no family-name alias is registered for that family. | Use `arkcli models get <name> --transform 'primary_version'` to get the version number, concatenate it into `<name>-<primary_version>`, and then pass it. |

## Direct `arkcli api` call (raw)

The operations behind `+chat` are `arkruntime.create_responses` and `arkruntime.create_responses_stream`, which can be called directly:

```bash
# Plain text (minimal shape)
arkcli api arkruntime.create_responses --params '{
  "model":"dola-seed-2-1-turbo-260628",
  "input":"hello"
}'

# Message shape with role (content can be either string or list)
arkcli api arkruntime.create_responses --params '{
  "model":"dola-seed-2-1-turbo-260628",
  "input":[{"role":"user","content":"hello"}]
}'

# Multimodal input (list-form content)
arkcli api arkruntime.create_responses --params '{
  "model":"dola-seed-2-1-turbo-260628",
  "input":[{
    "role":"user",
    "content":[
      {"type":"input_image","image_url":"file:///abs/path.jpg"},
      {"type":"input_text","text":"describe this"}
    ]
  }]
}'
```

Note: During raw API calls, **a local file `file://` URL depends on the SDK's automatic upload**, and the original path cannot be a relative path.

## Relationship with other commands

- **[`arkcli +gen`](../../arkcli-gen/SKILL.md)**: Used for image/video **generation**, not chat.
- **[`arkcli models`](../../arkcli-models/SKILL.md)**: Used to look up model names and modalities. Use it first when you cannot find a suitable model.

## Security and privacy

- `--input @<file>` uploads the file **in full** to the ARK file service (handled automatically by the SDK).
- Do not upload sensitive files casually. Uploaded files are retained on the server for a period of time. Do not assume they "disappear right after upload".
- When using `--debug`, network request content is printed to stderr. Redact the API key before sending logs externally.