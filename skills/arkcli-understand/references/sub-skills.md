# +understand sub-skill recipe catalog

> **Prerequisite:** Read [`arkcli-understand.md`](arkcli-understand.md) first (commands/flags/returns/errors).

The 12 sub-skills are 12 recipes `{modality, default model, built-in system prompt}` running on the same Responses engine. The "expected output" below comes from each sub-skill's built-in system prompt and guides the Agent when parsing `.content`.

General rules:

- **Do not override the default model** (verified working). Built-in catalog names are resolved to their current primary versions. A replacement `--model` must use a complete versioned ID or `ep-xxx`.
- The `[prompt]` positional argument is an **additional user requirement** layered on the expert prompt (fields, target, language, granularity, and others).
- To change output format/language while preserving expert capability, use `--system-prompt-append`; for complete customization, use `--system-prompt-override`.
- JSON outputs (grounding / gui / doc-extract) are strings in `.content` and require secondary parsing with `jq -r .content | jq .`.

---

## Image modality (default model `seed-2-0-lite`)

### `image-caption` — Image description / OCR
- **Purpose**: Clear, structured description and analysis of one or multiple images; for OCR, fully preserve image text without paraphrasing.
- **Expected output**: Natural-language description; verbatim text for OCR.
- **Examples**:
  ```bash
  arkcli +understand image-caption --input @photo.jpg "Describe the main subject and scene"
  arkcli +understand image-caption --input @receipt.png "Perform OCR and preserve all text and numbers exactly"
  arkcli +understand image-caption --input @a.jpg --input @b.jpg "Compare the differences between these two images"
  ```
- **Automatic routing**: When sub-skill is omitted and the first `--input` is an image, falls back to this sub-skill.

### `image-grounding` — Visual localization (bbox)
- **Purpose**: Given an image and target description, precisely locate targets in the image.
- **Expected output**: bbox coordinates `(x1, y1, x2, y2)` + confidence for every target. If the target does not exist, output empty array `[]`; do not fabricate.
- **Examples**:
  ```bash
  arkcli +understand image-grounding --input @scene.jpg "Draw bounding boxes around all red vehicles"
  arkcli +understand image-grounding --input @ui.png "Locate the login button" | jq -r .content | jq .
  ```

### `image-gui` — GUI action recognition
- **Purpose**: Given a UI screenshot and operation goal, output an executable action sequence for a GUI agent.
- **Expected output**: JSON array. Each step includes action type, target-element bbox where applicable, and required parameters such as typed text. Actions are limited to 12 types: `click` / `double-click` / `right-click` / `tap` / `long-press` / `scroll` / `swipe` / `drag` / `type` / `key` / `wait` / `screenshot`.
- **Example**:
  ```bash
  arkcli +understand image-gui --input @screen.png "Enter arkcli in the search box and click Search"
  ```

---

## Video modality

### `video-summary` — Video summary (default `seed-2-0-lite`)
- **Purpose**: Structurally summarize video using visuals + audio.
- **Expected output**: User-selected granularity: `overall` / `segment` / `chapter`; each segment includes key timestamps (`HH:mm:ss`), visual points, dialogue points, and overall topic. Do not fabricate absent content.
- **Example**:
  ```bash
  arkcli +understand video-summary --input @lecture.mp4 "Summarize by chapter and provide the start timestamp for each chapter"
  ```

### `video-qa` — Video Q&A (default `seed-2-0-lite`)
- **Purpose**: Answer a specific user question accurately using visuals / audio / timeline.
- **Expected output**: Direct answer first, with supporting evidence (visual description / dialogue / timestamp) when needed. If absent, clearly state that it is not covered; do not fabricate.
- **Example**:
  ```bash
  arkcli +understand video-qa --input @clip.mp4 "How many people appear in the video, and what is each person doing?"
  ```
- **Automatic routing**: When sub-skill is omitted and the first `--input` is video, falls back to this sub-skill.

### `vau` — Joint audiovisual understanding (default `seed-2-0-lite`)
- **Purpose**: VAU (Video-Audio Understanding), deeply analyzes audio elements (voice / effects / music), acoustic characteristics, and narrative roles. Registered as video; **input remains a video file**.
- **Expected output**: Clearly structured, detailed audio analysis report.
- **Example**:
  ```bash
  arkcli +understand vau --input @film_clip.mp4 "Analyze the sound design in this video"
  ```
- **Note**: `vau` is not the default video route (`video-qa` is); specify it **explicitly**.

---

## Audio modality (default model `seed-2-0-lite`)

> Audio is inlined as base64 with a **25 MB limit**. See [`arkcli-understand.md`](arkcli-understand.md).

### `asr` — Speech transcription
- **Purpose**: Pure ASR speech-to-text.
- **Expected output**: **Only** transcription text: no introduction, explanation, or Markdown. Output an empty string for unclear/no speech.
- **Examples**:
  ```bash
  arkcli +understand asr --input @speech.mp3
  arkcli +understand asr --input @speech.mp3 | jq -r .content   # Retrieve transcription text directly
  ```
- **Automatic routing**: When sub-skill is omitted and the first `--input` is audio, falls back to this sub-skill.

### `asr-align` — Subtitle alignment
- **Purpose**: Timestamped transcription producing subtitles.
- **Expected output**: **SRT** by default (index / start-end `HH:MM:SS,mmm --> HH:MM:SS,mmm` / text, with blank lines between entries); follow a user template if supplied.
- **Examples**:
  ```bash
  arkcli +understand asr-align --input @speech.mp3                       # Default SRT
  arkcli +understand asr-align --input @speech.mp3 "Output in WebVTT format"   # Custom template
  ```

### `asr-speakers` — Multi-speaker transcription
- **Purpose**: Transcribe multi-person dialogue and label speakers.
- **Expected output**: Speaker-labeled transcription: first speaker `[spk0]`, second `[spk1]`, and so on.
- **Example**:
  ```bash
  arkcli +understand asr-speakers --input @meeting.wav
  ```

### `ast` — Speech translation
- **Purpose**: Translate spoken audio content into text (AST).
- **Expected output**: Translated text. The built-in prompt does not fix the target language; specify it with `[prompt]`, such as "translate into English".
- **Example**:
  ```bash
  arkcli +understand ast --input @speech.m4a "Translate into English"
  ```

### `meeting-minutes` — Meeting minutes
- **Purpose**: Produce structured minutes from meeting audio (the built-in prompt also supports meeting-video semantics).
- **Expected output**: Markdown minutes containing meeting topic / participants (voiceprints labeled `spk0`, `spk1`, and others) / agenda and discussion points (chronological with timestamps) / decisions (Action Items with owner / deadline when identifiable) / follow-ups. Preserve key quotations and avoid subjective inference.
- **Example**:
  ```bash
  arkcli +understand meeting-minutes --input @standup.m4a
  ```

---

## File modality (default model `seed-2-0-mini`)

### `doc-extract` — Document field extraction
- **Purpose**: Extract structured information from PDF/documents.
- **Expected output**: JSON following the user-specified schema (field names / meanings). Use `null` when absent; do not invent. Preserve source page number `page` for traceability in multi-page documents.
- **Examples**:
  ```bash
  arkcli +understand doc-extract --input @invoice.pdf "Extract: invoice number / amount / invoice date / seller"
  arkcli +understand doc-extract --input @contract.pdf "Extract Party A, Party B, contract amount, and signing date" | jq -r .content | jq .
  ```
- **Automatic routing**: When sub-skill is omitted and the first `--input` is a non-image/video/audio file such as `.pdf`, falls back to this sub-skill.
