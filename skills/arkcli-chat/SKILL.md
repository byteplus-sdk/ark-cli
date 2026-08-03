---
name: arkcli-chat
version: 1.1.1
description: "arkcli +chat: Quickly chat/reason through the data-plane Responses API. Supports multimodal input (@file local images, videos, audio, and general files), streaming output, system instructions, sampling controls (temperature/top-p/max-output-tokens), reasoning effort controls, and multi-turn continuation with --store + previous-response-id. Use this when users need to chat with the model in real time, ask questions, reason, have open-ended multimodal conversations and follow-ups with images/videos/audio, adjust temperature/sampling, control thinking intensity, or continue a multi-turn conversation. Note: For multimodal understanding with a clear output form, such as speech transcription, document parameter extraction, subtitle timestamping, bbox selection and grounding, or video chapter summaries, use arkcli-understand. This skill is only for open-ended chat/reasoning."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli +chat --help"
---

# arkcli +chat

**CRITICAL — Before starting, MUST first use the Read tool to read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md), which contains the authentication gate, configuration troubleshooting, and command selection order.**
**CRITICAL — Before running `+chat`, be sure to first use the Read tool to read [`references/arkcli-chat.md`](references/arkcli-chat.md). Do not blindly call the command directly.**

## Core concepts

- When `--model` is omitted, the CLI automatically falls back to `resources.text.default` in the active profile. If the user explicitly passes `--model X` and `X` is not equal to that default, ask the user whether to promote it according to "Default drift detection and promote nudge" in [`../arkcli-shared/references/profile-defaults.md`](../arkcli-shared/references/profile-defaults.md).
- `+chat` is a high-level wrapper for the data-plane Responses API (`POST /responses`): one request returns the assistant text.
- Supports **multimodal** input: use `--input @photo.jpg` to upload a local file with the request, so the model can answer after viewing images/videos or listening to audio. Images (`.jpg/.png/.webp/...`), videos (`.mp4/.mov/...`), audio (`.mp3/.wav/.m4a/...`), and general files are automatically routed by extension.
- Supports **streaming**: `--stream` mode outputs incrementally, with reasoning (thinking) first, then the final answer (response).
- Supports **progress prompts**: For non-streaming calls, after 5 seconds, heartbeat lines like `arkcli +chat: still running… elapsed Xs` are output to stderr every 10 seconds to avoid long calls showing no output. In scripts, you can disable this with `--no-progress`. stdout is not affected.
- Supports **system instructions**: `--instructions "..."` Injects system-level instructions.
- Supports **sampling controls**: `--temperature` / `--top-p` / `--max-output-tokens`.
- Supports **thinking intensity**:`--reasoning-effort minimal|low|medium|high` (only takes effect on models that support reasoning).
- Supports **multi-turn continuation**: `--store` persists the current response, and the next call continues with `--previous-response-id <id>`.
- Supports **Tools**: `--tools mcp` (simple syntactic sugar) or `--tools-file tools.json` (full function form), together with `--tool-choice auto|required|none` and `--max-tool-calls`. See [`references/tools.md`](references/tools.md).
- Supports **reconciliation-plane commands**: `arkcli chat get/delete/list-input-items <response-id>` performs CRUD on responses stored with `--store`. See [`references/chat-meta.md`](references/chat-meta.md).
- Supports **caching and thinking**:`--caching enabled|disabled` (with `--cache-prefix`) controls the server-side prompt cache.`--thinking auto|enabled|disabled` controls the thinking stage. `--expire-at <epoch_sec>` adds an expiration time to a stored response. See [`references/caching-thinking.md`](references/caching-thinking.md).
- Supports **streaming events**: `--stream --include-events` outputs raw SDK events as NDJSON (one JSON per line) for autotest / agent programmatic consumption. See [`references/stream-events.md`](references/stream-events.md).
- Supports **verifiable strict JSON**: `--text-format json_schema --text-schema <file> --text-strict` checks locally that the response is completed, is a direct JSON value, and matches the schema. Otherwise the command exits non-zero. Strict streaming buffers events until validation succeeds so partial JSON is never emitted. See [`references/text-format.md`](references/text-format.md).
- **Output schema (be sure to read first)**: `+chat` returns the arkcli **flat** schema (`{id, model, content, reasoning_content, usage, ...}`), **not** the native Responses API nested `output[].content[].text` structure. Get assistant text directly from `.content`.`--format` does not switch the shape. See the "Return values" section in [`references/arkcli-chat.md`](references/arkcli-chat.md).

## Quick decision

**Criteria for using `+chat` (all three must be met):**
1. The user provides images/videos/audio, but the intent is **open-ended chat/question answering/reasoning/thoughts/comments** (not a fixed output form).
2. Cannot be mapped to one of the 12 `arkcli-understand` subskills.
3. Or the user needs multi-turn continuation with `--store` / `--previous-response-id`.

**Criteria for switching to `arkcli-understand` (switch if any one is met):**
- The user wants "transcription/speech-to-text/speech recognition/ASR" → understand.
- The user wants "subtitles/timestamping/SRT" → understand.
- The user wants "multi-speaker dialogue/speaker labeling/meeting transcription" → understand.
- The user wants "OCR/text recognition/recognize text in images/invoice amount" → understand.
- The user wants "draw a box/mark a box/bbox/visual grounding" → understand.
- The user wants "PDF parameter extraction/document extraction/extract parameters/contract extraction" → understand.
- The user wants "video summary/segmentation/chapter summary" → understand.
- The user wants "video Q&A/questions about video content" → understand.

In short: **If there is an @file and a clear output form → understand; image-based chat/follow-up/open-ended thoughts → chat**.

- If the user wants to generate images/videos (not chat), switch to [`../arkcli-gen/SKILL.md`](../arkcli-gen/SKILL.md).

## Agent quick execution order

1. When the user's goal is chat/reasoning/multimodal understanding, prefer `arkcli +chat`.
2. When the authentication status is uncertain, check `arkcli auth status` first. If not logged in or no API key is available, switch to [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).
3. When the model name is uncertain, switch to [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md) first.
4. **`--model` must be in the full `<name>-<primary_version>` form** (or Endpoint ID `ep-xxx`). The `primary_version` format is not fixed: it can be a 6-digit date, 8-digit date, qualified prefix, short number, or even an empty string (see the full table in link 0 of [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md) for details). **Do not use your own regex to guess whether it "looks complete".**If the user only provides a family name or it is unclear whether the name is complete, first look up `primary_version` and then concatenate it. If you just ran `models search/list`, directly reuse the parameter returned there. Otherwise, use `arkcli models get <name> --transform 'primary_version' | tr -d '"'` (`--transform` outputs quotes, which must be removed). Skipping this will directly cause a 404 `InvalidEndpointOrModel.NotFound`.
5. Add `--stream` when streaming output is needed. Add `--input @<file>` (can be repeated) when multimodal input is needed.

## Common fallbacks

- If the model name is uncertain, run `models search` first.
- To use an endpoint ID with `+chat`, directly pass `--model ep-xxx` (the endpoint itself already determines the modality, so no extra flag is needed).
- If authentication fails, switch to [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).

## Command overview

| Command |Description|
|------|------|
| `arkcli +chat --model <id> "<prompt>"` | Basic usage: plain-text chat |
| `arkcli +chat --model <id> --stream "<prompt>"` | Streaming output (two parts: thinking + response) |
| `arkcli +chat --model <id> --instructions "You are a concise assistant" "<prompt>"` | System-level instructions |
| `arkcli +chat --model <id> --temperature 0.2 --max-output-tokens 256 "<prompt>"` | Sampling controls |
| `arkcli +chat --model <id> --reasoning-effort high "<prompt>"` | Increase thinking intensity |
| `arkcli +chat --model <id> --input @file.jpg "<prompt>"` | Multimodal (local files, supports images/videos/audio) |
| `arkcli +chat --model <id> --input @a.jpg --input @b.jpg "<prompt>"` | Multiple files |
| `arkcli +chat --model <id> --store "<prompt>"` Get the id, then use `--previous-response-id <id> "<next sentence>"` | Persistence + multi-turn continuation |
| `arkcli +chat --model <id> --tools mcp --tool-choice auto "<prompt>"` | Tools: MCP |
| `arkcli +chat --model <id> --tools-file tools.json --tool-choice required "<prompt>"` | Tools: custom function |
| `arkcli chat get <response-id>` | Retrieve a stored response (including function_calls) |
| `arkcli chat list-input-items <response-id> --order desc --limit 5` | List input items (multi-turn history) |
| `arkcli chat delete <response-id>` | Delete a stored response |
| `arkcli +chat --model <id> --caching enabled --store "<prompt>"` | Enable prompt cache + persistence |
| `arkcli +chat --model <id> --thinking disabled --max-output-tokens 100 "<prompt>"` | Disable thinking to shorten output |
| `arkcli +chat --model <id> --text-format json_object "<prompt>"` | Force the model to output valid JSON |
| `arkcli +chat --model <id> --text-format json_schema --text-schema schema.json --text-strict "<prompt>"` | Strongly constrain the shape with JSON Schema |
| `arkcli +chat --model <id> --stream --include-events "<prompt>"` | Streaming NDJSON (one SDK event JSON per line) |

## Detailed documentation

- For all `+chat` parameters, return values, error codes, the automatic multimodal file upload mechanism, and more, see [`references/arkcli-chat.md`](references/arkcli-chat.md).
- For `+chat` Tools capabilities (`--tools / --tools-file / --tool-choice / --max-tool-calls`, including function and mcp), see [`references/tools.md`](references/tools.md).
- For the three reconciliation-plane commands `chat get / chat delete / chat list-input-items`, see [`references/chat-meta.md`](references/chat-meta.md). The responses operated on by these commands must have been stored with `+chat --store`.
- For `+chat` usage of `--caching / --cache-prefix / --thinking / --expire-at`, echoed parameters, and autotest mappings, see [`references/caching-thinking.md`](references/caching-thinking.md).
- For `+chat` usage of `--text-format / --text-schema / --text-schema-name / --text-strict`, see [`references/text-format.md`](references/text-format.md).
- For `+chat` streaming event output with `--stream --include-events` NDJSON, see [`references/stream-events.md`](references/stream-events.md).
