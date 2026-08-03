# Minimum arkcli-understand evaluation cases

Goal: verify stable trigger, automatic routing, anti-trigger, write-operation, and privacy-guard behavior, while preventing common hallucinations (nonexistent flags, command spelling, and incorrect JSON paths).

## 1) Trigger: understanding task with a specific output form

Input:

- "Transcribe this recording `@interview.mp3`."
- "Draw boxes around everyone in `@scene.jpg` and return coordinates."

Expected behavior:

- Route to `arkcli-understand`.
- Audio transcription → `arkcli +understand asr --input @interview.mp3`.
- Bounding-box localization → `arkcli +understand image-grounding --input @scene.jpg "Draw bounding boxes around everyone"`; explain output bbox `(x1,y1,x2,y2)` + confidence and secondary parsing with `jq -r .content | jq .`.
- **Do not** use `+chat` for these fixed-output tasks.

## 2) Automatic routing: omit sub-skill

Input:

- "Understand what this audio `@speech.mp3` says" (no sub-skill named).

Expected behavior:

- `arkcli +understand "<prompt>" --input @speech.mp3`; explain automatic routing by audio modality to `asr`.
- If the user truly wants translation / subtitles / multiple speakers, explain these are **not** selected automatically and require explicit `ast` / `asr-align` / `asr-speakers`.

## 3) Anti-trigger: open-ended image conversation

Input:

- "Look at `@cat.jpg`, chat with me about it, and let me ask follow-up questions."

Expected behavior:

- Route to [`arkcli-chat`](../../arkcli-chat/SKILL.md): open conversation + multi-turn continuation (`--store` / `--previous-response-id`) belongs to `+chat`.
- **Do not** force `+understand` (it is single-turn specialized understanding without `--previous-response-id`).

## 4) Anti-trigger: generation, not understanding

Input:

- "Generate a cyberpunk cat image."

Expected behavior:

- Route to [`arkcli-gen`](../../arkcli-gen/SKILL.md).
- **Do not** use `+understand` (it understands existing material; it does not generate).

## 5) Guard: large audio / private upload

Input:

- "Transcribe `@2hours-meeting.wav` (about 300 MB)."

Expected behavior:

- First state that audio uses base64 inline with a **25 MB limit**. Oversized files are not blocked but will likely fail; recommend splitting / lowering bitrate / transcoding. **Do not** pretend that a large file will work directly.
- Explain that `--input` uploads/inlines the complete file and sensitive material requires user confirmation.

## 6) Guard: do not retry data-plane authentication failure

Input (command response):

- `Error code: 403 - {"code":"AccessDenied",...}` or `ark runtime: API Key is required`

Expected behavior:

- **Do not retry in place**. Follow "API Key mode error recovery" in [`../../arkcli-auth/references/auth-modes.md`](../../arkcli-auth/references/auth-modes.md), guide the user to run `arkcli auth apikey`, select a key, and return to the original task.

## 7) Agent anti-hallucination checklist

Any of the following from the Agent is a failure:

- `arkcli understand ...` (missing `+`).
- `arkcli +understand ...` without `--input` (unless demonstrating an error).
- Using `--instructions` with `+understand` (it has no such flag; use `--system-prompt-override` / `--system-prompt-append`).
- Using `--tools` / `--tools-file` / `--text-format` / `--previous-response-id` / `--include-events` with `+understand` (these are `+chat` flags).
- Treating built-in catalog default `seed-2-0-lite` as a value sent directly to the data plane and manually adding a version. arkcli resolves the built-in name to its current primary version; a complete versioned ID is required **only when the user explicitly overrides `--model`**.
- Parsing with `jq '.output[].content[].text'` (output is flat; use `jq -r .content`, then `| jq .` for JSON output).
- Inventing nonexistent sub-skill names. The only valid names are `image-caption` / `image-grounding` / `image-gui` / `doc-extract` / `video-summary` / `video-qa` / `vau` / `asr` / `asr-align` / `asr-speakers` / `ast` / `meeting-minutes`.
- Treating `vau` as automatic video routing default (the default is `video-qa`; `vau` must be explicit).

## 8) Happy path (end-to-end; prerequisites required)

Prerequisite: logged in according to `arkcli auth status` and ARK API Key configured.

```bash
# Image description (easiest to reproduce)
arkcli +understand image-caption --input @<local-image> "Describe in one sentence" | jq -r .content
```

Check: exit code 0; `.content` is non-empty and describes the image. Run one corresponding happy path for each remaining modality (audio ≤25 MB).

## 9) Supporting machine evaluation

Machine-evaluation assets should be stored in `tests/skills/arkcli-understand/` (not under `skills/`). Rerun:

```bash
cd skill-creator
python3 -m scripts.run_arkcli_skill_benchmark \
  --skill-path ../skills/arkcli-understand \
  --workspace /tmp/arkcli-understand-bench \
  --iteration 1 \
  --runs-per-config 2 \
  --runtime claude
```
