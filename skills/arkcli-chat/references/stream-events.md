---
name: stream-events
description: arkcli +chat --include-events usage reference. In streaming mode, outputs raw SDK events as NDJSON for programmatic consumption by autotest / agents.
---

# +chat Stream Events (--include-events)

In streaming mode, outputs each SDK event as one line of JSON (NDJSON), instead of human-readable `Thinking:` / `Response:` text.

## When to use

- **autotest** —— Automated tests need per-event assertions (event type, usage, response ID, etc.).
- **agent / programmatic consumption** —— Upstream programs need to parse streaming output as structured data, instead of extracting it from human-readable text with regular expressions.
- **Debugging** —— Use this when you want to see each event type and parameter actually sent by the server.

## Flag quick reference

| flag | Purpose |
|---|---|
| `--include-events` | Streaming mode only; when enabled, stdout outputs NDJSON (one raw SDK event JSON per line) |

`--include-events` **must be used with `--stream`**; this flag has no effect in non-streaming mode.

## Typical usage

### Minimal: streaming NDJSON

```bash
arkcli +chat "Explain quantum entanglement" --model ep-xxx --stream --include-events
```

Output example (one independent JSON object per line):

```json
{"type":"response.created","response":{"id":"resp_abc123","object":"response","status":"in_progress",...}}
{"type":"response.reasoning_summary_text.delta","delta":"Quantum entanglement is..."}
{"type":"response.output_text.delta","delta":"In simple terms,"}
{"type":"response.output_text.delta","delta":"two particles..."}
{"type":"response.completed","response":{"id":"resp_abc123","status":"completed","usage":{"input_tokens":10,"output_tokens":50,"total_tokens":60}}}
```

### Filter specific events with jq

```bash
# Show only delta events
arkcli +chat "Write a short poem" --model ep-xxx --stream --include-events | jq 'select(.type | contains("delta"))'

# Show only the final completed event to get usage
arkcli +chat "What is 1+1?" --model ep-xxx --stream --include-events | jq 'select(.type == "response.completed")'

# Extract all delta text and concatenate it into the full answer
arkcli +chat "Introduce Beijing" --model ep-xxx --stream --include-events | jq -r 'select(.type | contains("output_text.delta")) | .delta' | tr -d '\n'
```

### Use with --store + chat get

```bash
# After the streaming run finishes, get the ID from the completed event.
RESP_ID=$(arkcli +chat "Remember that my name is Jim" --model ep-xxx --store --stream --include-events \
  | jq -r 'select(.type == "response.completed") | .response.id' | head -1)

# Later, use chat get to query the full response
arkcli chat get "$RESP_ID" --format json | jq '{id, content, usage}'
```

## Difference from `--format json`

| Dimension | `--stream --include-events` | `--stream` (without include-events) | `--format json` (non-streaming) |
|---|---|---|---|
| Output format | NDJSON (one event per line) | Human-readable text | Single JSON object |
| Event granularity | Each raw SDK event | Outputs only text delta | Output after aggregation |
| Parsability | High (independent JSON per line) | Low (requires regular expressions) | High (single JSON) |
| Real-time behavior | Real-time per event | Real-time per text segment | Waits until everything is complete |
| Purpose | Autotest / agent consumption | Human reading | Get complete results in a structured format |

## ChatStreamDelta event types

When `--include-events` is enabled, `ChatStreamDelta` at the service layer carries the following metadata parameters:

| Parameter |Description|
|---|---|
| `EventType` |Type of the SDK event (such as `response.output_text.delta`)|
| `RawEvent` | Raw event JSON string (that is, each line in the NDJSON output) |
| `Content` | Text delta (text delta events only) |
| `ReasoningContent` | Reasoning delta (reasoning delta events only) |
| `FunctionCallArgs` | Tool call argument delta (function_call.arguments.delta events only) |
| `FinalResponseID` | Response ID (response.created / response.completed) |
| `FinalUsage` | Final usage (response.completed) |
| `FinalContent` | Full text (output_text.done) |
| `FinalReasoning` | Full reasoning text (reasoning_summary_text.done) |
| `Incomplete` | Whether it is incomplete (response.incomplete) |

## Common errors

| Symptom | Cause |
|---|---|
| `--include-events` has no output | `--stream` is not included; this flag takes effect only in streaming mode. |
| Output is plain text instead of JSON | `--include-events` is not included, so it falls back to human-readable mode. |
| The number of NDJSON lines is lower than expected. | Some event types (such as `response.output_item.added`) are recognized only in PR-4; the current version decodes only common events. |
| `jq` parsing error | Some lines may not be valid JSON (such as the `[DONE]` terminator); use `jq -R 'fromjson?' ` for fault tolerance. |

## Autotest unlock checklist

| Test case | Unlocked |
|---|---|
| `Test_Stream_IncludeEvents` | ✅ |
| `Test_Stream_EventTypeCoverage` (11 event types) | ✅ |
| `Test_Stream_NDJSONFormat` (independent JSON per line) | ✅ |
| `Test_Stream_CompletedEventHasUsage` | ✅ |
| `Test_Stream_TextFormat` (streaming input parameters can be passed; echo uses --include-events) | ✅ |