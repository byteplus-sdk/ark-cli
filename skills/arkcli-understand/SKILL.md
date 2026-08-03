---
name: arkcli-understand
version: 1.0.1
description: "arkcli +understand: a multimodal understanding workflow with 12 task-oriented sub-skills across image/video/audio/file modalities on the data-plane Responses API engine. Covers image captioning/OCR, bbox grounding, GUI action recognition, PDF/document field extraction, video summary/QA, joint audio-video understanding, ASR, speech translation, SRT alignment, speaker diarization, and meeting minutes. Platform profiles use their deployed Resources.Text.Default endpoint; plan profiles use each recipe's fallback model. Invoke for multimodal tasks with a specific output form, such as transcribing audio, locating objects, extracting PDF fields, summarizing video, generating subtitles, producing meeting minutes, or describing an image. Use +chat for open-ended visual conversations and +gen for image/video generation."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli +understand --help"
---

# arkcli +understand

**CRITICAL — Before starting, you MUST use Read to open [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md), which contains authentication gates, API Key error recovery, configuration troubleshooting, and command selection order.**
**CRITICAL — Before running `+understand`, you MUST use Read to open [`references/arkcli-understand.md`](references/arkcli-understand.md) (commands/flags/returns/errors) and [`references/sub-skills.md`](references/sub-skills.md) (purpose and expected output of all 12 sub-skills). Do not construct commands from memory.**

## Core concepts

- `+understand` is **not 12 implementations, but one engine plus a semantic layer**. The underlying engine is the same data-plane Responses API used by `+chat`; each sub-skill is a **recipe** `{modality, fallback model, built-in system prompt}`. When `--model` is omitted, a Platform Profile must use its deployed `Resources.Text.Default` endpoint; only plan profiles use the recipe fallback model.
- Command form: `arkcli +understand <sub-skill> --input @file [prompt]`.
  - If `args[0]` **matches the registry** (one of 12), it is the explicit sub-skill; remaining positional arguments become a prompt layered on the built-in prompt.
  - If `args[0]` **does not match**, all positional arguments become the prompt, and the server automatically routes by the modality of the **first `--input` file** to that modality's default sub-skill.
- **`--input` is required**: a sub-skill understands a particular file. Without `--input`, it immediately returns `missing_input`. A prompt alone cannot derive a recipe.
- Return values are identical to `+chat`: arkcli's **flat** schema `{id, model, content, reasoning_content, usage}`, **not** the native nested Responses `output[].content[].text`. See "Return value" in [`references/arkcli-understand.md`](references/arkcli-understand.md).
- Multimodal upload: image/video/doc use the SDK `file://` preprocessor and Files API automatically. **Audio is special**: it is inlined as a base64 data URL, with a **25 MB limit**. See the reference.
- `--dry-run` resolves the local recipe, the active Profile's persisted default endpoint, an explicit model, and input-reference data and emits `preview.v1`. It never reads live Endpoint/model metadata, uploads files, calls Responses, creates usage or response IDs, or persists a response. Online-only values must be listed as `unresolved`; the plan is not server validation.

## Quick decision: understand vs chat vs gen

| User intent | Route |
|---|---|
| Multimodal understanding task with a **specific output form** (transcription / translation / subtitles / bbox localization / GUI actions / field extraction / chapter summary / multiple speakers / meeting minutes) | **`+understand`** (a matching sub-skill) |
| Open-ended image/video conversation, follow-up questions, reasoning, multi-turn continuation (`--store`/`--previous-response-id`), tools (web_search/function), or strict JSON with `--text-format json_schema` | [`../arkcli-chat/SKILL.md`](../arkcli-chat/SKILL.md) |
| **Generate** images/videos rather than understand existing material | [`../arkcli-gen/SKILL.md`](../arkcli-gen/SKILL.md) |

One-line criterion: **if the task maps to a sub-skill below, use `+understand`; otherwise use `+chat` for open conversation and `+gen` for generation.**

## Agent quick execution order

1. Determine which sub-skill matches the user's task. If matched, specify the sub-skill explicitly instead of relying on automatic routing.
2. Pass the authentication gate: if login state is uncertain, run `arkcli auth status`. Data-plane calls use an ARK API Key (`Authorization: Bearer`). On authentication failure, follow "API Key mode error recovery" in [`../arkcli-auth/references/auth-modes.md`](../arkcli-auth/references/auth-modes.md), run `arkcli auth apikey`, and **do not retry in place**.
3. Prepare input: `--input @<file>` (repeatable). Local files use `@`; remote input uses `https://` / `tos://`.
4. **Do not pass `--model` by default**: a Platform Profile uses the deployed `ep-xxx` in `Resources.Text.Default`, consistently with `+chat`; a plan profile uses the sub-skill's recipe fallback model. Only pass `--model` when the user explicitly requests another resource; it must be a complete versioned ID or `ep-xxx`. If uncertain, route to [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md).
5. Add `--stream` for incremental output. To adjust instructions, use `--system-prompt-append "..."` (append) or `--system-prompt-override "..."` (replace the built-in prompt).
6. Return to the user's original goal after execution; do not stop at an intermediate artifact.

## Sub-skill quick reference (12 / 4 modalities)

> The models below are recipe fallbacks for plan profiles. A Platform Profile does not invoke these bare names; it uses the Profile's deployed `Resources.Text.Default` endpoint. See Agent step 4 above when an override is required.

| sub-skill | Modality | Plan recipe fallback model | Purpose |
|---|---|---|---|
| `image-caption` | image | `seed-2-0-lite` | Single/multiple image descriptions and OCR (preserve source text; do not paraphrase) |
| `image-grounding` | image | `seed-2-0-lite` | Visual localization: output target bbox `(x1,y1,x2,y2)` + confidence |
| `image-gui` | image | `seed-2-0-lite` | GUI screenshot → executable action sequence |
| `doc-extract` | file | `seed-2-0-mini` | Extract structured JSON from PDF/doc by schema (missing values null; preserve page) |
| `video-summary` | video | `seed-2-0-lite` | Video summary: overall / segment / chapter + key timestamps |
| `video-qa` | video | `seed-2-0-lite` | Video Q&A using visuals + audio + timeline |
| `vau` | video | `seed-2-0-lite` | Joint audiovisual understanding and audio-analysis report |
| `asr` | audio | `seed-2-0-lite` | Speech transcription: plain text only, with no prefix/suffix/formatting |
| `asr-align` | audio | `seed-2-0-lite` | Subtitle alignment: SRT by default (index + start/end + text) |
| `asr-speakers` | audio | `seed-2-0-lite` | Multi-speaker transcription with `[spk0]` / `[spk1]` labels |
| `ast` | audio | `seed-2-0-lite` | Speech translation into text |
| `meeting-minutes` | audio | `seed-2-0-lite` | Structured Markdown minutes (topic/participants/key points/decisions/follow-ups) |

See [`references/sub-skills.md`](references/sub-skills.md) for expected output and commands for each.

## Automatic routing when sub-skill is omitted

When `args[0]` does not match the registry, infer modality from the **first `--input` file extension** and fall back to its default sub-skill:

| Inferred modality | Fallback sub-skill |
|---|---|
| image (`.jpg/.png/.webp/...`) | `image-caption` |
| video (`.mp4/.mov/...`) | `video-qa` |
| audio (`.mp3/.wav/.m4a/...`) | `asr` |
| file (other extensions) | `doc-extract` |

- Without `--input`, automatic routing is impossible (a prompt alone cannot derive a recipe) → `missing_sub_skill`.
- Routing currently uses only file modality, not prompt keywords. For example, "draw a box around" does not automatically select `image-grounding`. Explicitly provide non-default sub-skills such as grounding / gui / summary / subtitles / translation.

## Command overview

| Command | Description |
|---|---|
| `arkcli +understand image-caption --input @photo.jpg "Describe this image"` | Image description |
| `arkcli +understand image-caption --input @doc.png "Perform OCR and preserve the text exactly"` | OCR |
| `arkcli +understand image-grounding --input @scene.jpg "Draw bounding boxes around all red cars"` | Visual bbox localization |
| `arkcli +understand doc-extract --input @invoice.pdf "Extract the invoice number, amount, and date"` | Document field extraction |
| `arkcli +understand video-summary --input @clip.mp4 "Summarize by chapter"` | Chapter-based video summary |
| `arkcli +understand video-qa --input @clip.mp4 "How many cars appear in the video?"` | Video Q&A |
| `arkcli +understand asr --input @speech.mp3` | Speech transcription (no prompt required) |
| `arkcli +understand asr-align --input @speech.mp3` | Generate SRT subtitles |
| `arkcli +understand asr-speakers --input @meeting.wav` | Multi-speaker transcription |
| `arkcli +understand ast --input @speech.m4a "Translate into English"` | Speech translation |
| `arkcli +understand meeting-minutes --input @meeting.m4a` | Meeting minutes |
| `arkcli +understand "<prompt>" --input @photo.jpg` | Omit sub-skill → automatic routing by modality (here → image-caption) |
| `arkcli +understand asr --input @speech.mp3 --stream` | Incremental streaming output |
| `arkcli +understand image-caption --input @x.jpg --system-prompt-append "Answer in English"` | Append instructions after the built-in prompt |
| `arkcli +understand image-caption --input @x.jpg "Describe the image" --dry-run` | Preview recipe/model/request summary without upload or inference |

## Detailed documentation

- All flags, `@file`/audio upload mechanisms, flat return schema, errors, raw API fallback, and relation to `+chat` → [`references/arkcli-understand.md`](references/arkcli-understand.md)
- Purpose, **expected output form**, and example commands for all 12 sub-skills → [`references/sub-skills.md`](references/sub-skills.md)
- Minimum evaluation/regression cases → [`references/evals.md`](references/evals.md)
