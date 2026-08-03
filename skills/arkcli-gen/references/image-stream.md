---
name: image-stream
description: arkcli +gen --stream streaming image output. One image_generation.* event JSON per line (NDJSON), suitable for incremental preview / visualizing progress / using jq to extract each frame URL.
---

# +gen --stream (image streaming)

By default, `+gen` waits synchronously until the entire generation finishes. `--stream` switches the CLI to NDJSON output and writes the server-side SSE streaming events to stdout as-is, one event per line. Suitable for:

- Multi-image generation scenarios (`--image-count >1`) where you want to process the first image as soon as it is available.
- Connecting to jq / script pipelines for incremental preview or parallel downloads.
- Debugging server-side streaming behavior.

## Scope

**Image tasks only**. Video tasks use the async task + polling model (`+gen` polls by itself) and have no SSE channel. Passing `--stream` with a video model will be intercepted by `+gen` before the request is sent:

```
--stream only applies to image generation tasks
hint: remove --stream (video tasks use async polling, not SSE),
      or pass --modality image with an image-capable model / endpoint
```

## Event protocol

The underlying implementation uses SDK `arkruntime.Client.GenerateImagesStreaming`, with three event types:

| Event type | When to send | Parameters |
|---|---|---|
| `image_generation.partial_succeeded` | When each image finishes generating | `url` / `b64_json` / `size` / `image_index` / `model` / `created` |
| `image_generation.partial_failed` | A single image fails to generate (does not necessarily terminate the stream). | `error.code` / `error.message` |
| `image_generation.completed` | Stream ends | Optional `usage{generated_images, output_tokens, total_tokens}` / `tools[]` |

The CLI outputs one `ImageStreamDelta` JSON per line (service-layer view). Key parameters:

```json
{
  "event_type": "image_generation.partial_succeeded",
  "model": "seedream-5-0-260128",
  "created": 1700000000,
  "image_index": 0,
  "url": "https://...",
  "size": "1024x1024"
}
```

Note:
- For `partial_failed`, `error.code/message` are two flat parameters in ImageStreamDelta: `error_code` / `error_message`.
- For the `completed` event, `usage` / `tools` are renamed to `final_usage` / `final_tools` in ImageStreamDelta.
- The `b64_json` (Base64 string image) parameter appears only when ResponseFormat is `b64_json` (default: `url`).

## Command quick reference

```bash
# 1. Stream-generate a single image (process it as soon as you see the first frame).
arkcli +gen "neon kitten" --model seedream-5-0-260128 --stream

# 2. Stream-generate 3 images and get the URLs in order.
arkcli +gen "5 different futuristic cars" \
  --model seedream-5-0-260128 \
  --image-count 3 --stream \
  | jq -r 'select(.event_type=="image_generation.partial_succeeded") | .url'

# 3. Streaming + decode Base64 (first switch b64_json to the stream).
arkcli +gen "blue laptop" \
  --model seedream-5-0-260128 --stream \
  | jq -r 'select(.b64_json != "") | .b64_json' \
  | base64 -d > out.png

# 4. Monitor error events + final usage.
arkcli +gen "..." --model ... --stream \
  | jq 'select(.event_type=="image_generation.partial_failed" or .event_type=="image_generation.completed")'
```

## Relationship with `--save-to`

`--save-to` is the "automatic download" feature on the synchronous path. `--stream` **does not trigger automatic downloads**.

Reason: the purpose of streaming is to let users process output incrementally. The CLI should not write files to disk in parallel and interfere. When you need both disk output and streaming monitoring, pipe the streaming output to your own download script (see example 3 above).

## Relationship with the raw API

`+gen --stream` is equivalent to:

```bash
arkcli api arkruntime.generate_images_stream \
  --params '{"model":"...","prompt":"...","stream":true,"response_format":"url"}'
```

However, the raw API cannot receive the ParseAssetRefs / collapseImageAssets / modality routing capabilities from the `arkcli +gen` layer. For normal use, use `+gen --stream`. Use the raw API only when debugging SDK parameters or testing server-side behavior.

## Scenarios where streaming is not supported

| Scenario | Why | Alternative |
|---|---|---|
| A model or Endpoint resolved as video (or explicit `--modality video`) | Video uses the async task + poll model and has no SSE channel. | Do not pass `--stream`; use the default polling of `+gen`. |
| Synchronously wait for accumulated results | In `--stream` mode, stdout is an NDJSON stream and does not output the accumulated GenerateOutput. | Do not pass `--stream`; use the default synchronous path. |

## Common errors

| Symptom | Cause |
|---|---|
| `--stream only applies to image generation tasks` | Structured metadata resolves the model/Endpoint as video, or `--modality video` is explicitly specified. |
| The first line of streaming output is `partial_failed`, followed by no further output. | The upstream request actually failed. Check the `error_code` parameter. This is especially common with small sizes such as `--size 1024x1024`. |
| The `completed` event is not shown. | When `--image-count 1`, the server may not send `completed`; the single image is the end. It is sent only for multiple images. |
