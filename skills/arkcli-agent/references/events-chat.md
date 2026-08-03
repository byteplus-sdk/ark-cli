# Events and Chat

## Event/Chat Experience

Choose the entry point based on whether the user expects an Agent reply:

| Scenario | Preferred Entry | Completion Requirement |
| --- | --- | --- |
| Create a Session and start a conversation | `arkcli +new session <agent-id> --message "..."` | The CLI follows events until a terminal state and prints the reply |
| Send a question to an existing Session | `agent session events send <session-id> --type user.message --text "..." --wait` | Wait for `agent.message` and `idle` or another terminal state |
| Large payload (about 50 KiB or more) or long-running research/report task | `agent session events send <session-id> ... --poll` | Poll `events list` without opening a stream; for even longer work, send without waiting and poll in separate calls |
| Deliver an event without waiting | `agent session events send ...` | Use only when the user explicitly requests asynchronous delivery or a script is writing events |
| Observe an existing Session in real time | `arkcli +tail <session-id>` | Continue until a terminal state or explicit error |

### Event Deltas

The data plane `/events/stream` endpoint accepts repeated query parameters for low-latency incremental frames. Real-time CLI entries request these by default:

```text
event_deltas=agent.message&event_deltas=agent.thinking
```

For these event types, the service may return an `event_start`, one or more `event_delta` frames whose text is in `delta.content[].text`, and then the complete `agent.message` or `agent.thinking` event. Delta frames refer to their start frame through `event_id`.

Pretty output for `+tail`, `events send --wait`, `+new session`, and `+iterate` labels incremental text as `[agent delta #N]` or `[thinking delta #N]`, then reports the final event once without printing the same content again. `--raw` preserves the original frames; the lower-level `agent session events stream` command emits raw NDJSON.

If a service deployment does not yet accept delta parameters, request complete events only:

```bash
arkcli +tail <session-id> --no-event-deltas
arkcli agent session events send <session-id> --text "..." --wait --no-event-deltas
arkcli agent session events stream <session-id> --no-event-deltas
```

`events list` is a history endpoint and never sends `event_deltas`. Stream recovery uses the last cursor to fetch persisted complete events, and the CLI deduplicates stream/list deliveries by event type, ID, and delta content.

**When a user asks an Agent to analyze, answer, or perform work, plain write-only `events send` is not a complete workflow. Use `--wait` for short requests, `--poll` for large or long-running requests, or follow an asynchronous send with cursor-based `events list` polling or `+tail`.**

- `agent session list --agent-id <agent-id>` filters by Agent; the CLI sends `AgentIds` array via `ListSessionsForTop` contract.
- Session lists are paginated with `--page` for `PageNumber` and `--limit` for `PageSize`. Use `--page-all` for global pagination and `--page-limit` to control the maximum number of pages.
- `events send` sends events actively. Common usage:

```bash
arkcli agent session events send <session-id> --type user.message --text "Help me analyze this data" --wait
```

- `+new session` and `+iterate` use a reliable event channel: consume `/events/stream`, compensate with `/events?order=asc&limit=100&after=...` after a disconnect or transport failure, then reconnect. Stream and list events are deduplicated by ID. Completion requires `idle`, `requires_action`, `failed`, `terminated`, or another terminal state; exhausting reconnect attempts is an error, not success.
- `+tail` uses the same channel for pretty and raw output. Pretty output truncates noisy user/thinking payloads while preserving event IDs and omitted-character counts; use `--raw` or `--max-event-chars 0` for full content.
- A successful `events send` response means the event was accepted, not that the Agent replied. `--wait` follows the returned cursor through the reliable channel. It streams for up to 120 seconds by default and then falls back to event-list polling for another 120 seconds; use `--wait-timeout` and `--wait-fallback-timeout` to adjust those stages.
- `--poll` skips streaming and polls the event list until a terminal state. It is useful for long work or unstable streaming connections. The default interval is two seconds and can be changed with `--poll-interval`.
- `--wait` and `--poll` are mutually exclusive. `--raw` keeps complete machine-readable events; `--max-event-chars` affects display only and never changes the payload sent to the Agent.
- For work that may exceed the current process timeout, send without `--wait/--poll`, retain the returned event cursor, and repeatedly run `events list --after <cursor> --order asc --limit 100 --format json`. Advance the cursor for every new event and stop only at a terminal state.
- On a polling timeout or connection interruption, retry the read with the same cursor. Stop immediately for authentication, authorization, validation, entitlement, or other explicit business errors. Never report a timeout as successful completion.
- Run each step as a separate invocation: create or identify the Session, send the event, extract the cursor, poll events, and optionally fetch final Session metadata. If a write times out, inspect `session get/list` or `events list` before deciding whether to retry it.

- `--events` or `--file` can provide multiple events. Currently, the online data plane API accepts a single event at a time, so the CLI sends them sequentially in array order; this order is not guaranteed to be atomic. The response marks `transport=serial`, `atomic=false`, and the number of sent events. If a middle retry fails, the error points to the failed `event[index]` and the number of sent events.
- Do not interpret this CLI compatibility as the server supporting atomic batches; for business requirements that need atomic submissions, wait for server support for batch requests.
- Multiple events can contain different `type`s, and the CLI does not reorder them. For example, sending `user.message` before `user.interrupt` makes the server start the corresponding turn before handling the interrupt; when checking, see the original `user.message`, `user.interrupt`, and interrupted model request. `user.interrupt` must be sent when an active turn exists; sending it for idle sessions produces `session.error: no active turn to cancel`.

- `events list` retrieves historical events with `--after/--before` event cursor, `--since/--created-after/--created-before` time filtering, and `--type` for repeated or comma-separated types. With `--page-all`, the default `limit` is `100`, and the response is merged by `next_page -> page` for `data`.
- `threads list` supports `--page-all` and `next_page -> page`. `resources list` does not have a pagination contract; do not fake page parameters.
- `events stream` outputs a human-friendly SSE data or NDJSON line.
- `/compact` recommendation: after sending, use `events list` or `+tail` to check `agent.thread_context_compacted`. If only the corresponding `user.message`, `thread_status_idle` are seen, but no compacted events, the CLI has correctly submitted the protocol; this should be investigated based on the server's capabilities, version, or routing; do not report this as "compressed".
- `+tail` outputs human-readable short lines by default, classifying them as `[user]`, `[agent]`, `[thinking]`, `[tool]`, `[tool_result]`, `[model]`, `[status]`, `[action]`, `[error]`, and `[outcome]`. For machine-readable output, use `+tail --raw`.
- `+debug <session-id>` reports `event_delta_count` and `event_delta_by_type` in addition to the event type summary, which helps verify whether the service actually returned incremental frames.
- `+chat <prompt>` retains the Responses API for quick dialogues, not entering Managed Agent.
- `+new session` is the Managed Agent session entry point (early PRD usage of `+chat` is understood as this command):
  - `arkcli +new session`: opens an interactive selector, first `ListSessions`, choose an existing session to enter REPL; or select `[New]` to `ListAgents`, `ListEnvironments`, and then `CreateSession` to enter REPL.
  - `arkcli +new session <agent-id> --environment-id <env-id>`: directly creates a new session and sends the first message or enters REPL.
  - TTY mode enters REPL.
  - Non-TTY stdin or `--message` is one-shot.
  - `/exit` exits, `/interrupt` sends `user.interrupt`.
  - `/compact` and `/clear` send `user.message` via slash envelope; do not judge backend completion based solely on successful transmission. Successful execution of `/compact` should see `agent.thread_context_compacted` in the event stream, and the service-side semantics are preserved; `/clear` follows the service-side event response. Both continue the REPL after completion.
  - `/allow [tool_use_id]` and `/deny [tool_use_id] [reason]` send tool confirmation.
- Scripts or non-interactive scenarios with known `session-id` should use `arkcli agent session events send <session-id>` or `arkcli agent session events stream <session-id>` or `arkcli +tail <session-id>` instead of the selector.
- `[All]` in the selector re-fetches sessions based on known states, ideally including terminated/archived sessions. If the backend introduces new state enumerations, they need to be updated synchronously.
- `[Create new agent]` in the selector currently redirects to `arkcli +new-agent "..."` without a natural language flow for agent creation within the selector.
- `events send` supports multi-modal convenient parameters:
  - `--image file-xxx` sends an image file source directly; `--image @./a.png` or `--image ./a.png` uploads the file via the Files API and waits for activation before sending.
  - `--document file-xxx` sends a document/PDF file source directly; `--document @./a.pdf` or `--document ./a.pdf` uploads the file via the Files API and waits for activation before sending.
  - Image content schema: `{type: image, source: {type: file, file_id: ...}}`.
  - Document/PDF content schema: `{type: document, source: {type: file, file_id: ...}}`.
  - `--image/--document` cannot be mixed with `--event/--events`; for manual event sending, the full content schema is used in `--event/--events`.
- Event-associated fields are validated by the CLI before sending, consistent with typed flags, `--event`, `--events`, and `--file` behaviors:
  - `user.custom_tool_result` must include `custom_tool_use_id` (typed flag: `--custom-tool-use-id`). It is not a global required field for all events but is semantically required for custom tool results; otherwise, the results cannot be associated with the specific custom tool call.
  - The service once incorrectly allowed events without this field and filled it with an empty string; do not rely on this compatibility. The CLI fails fast on requests without `custom_tool_use_id`, preventing the generation of orphaned events; the server should also reject such requests.
  - The CLI only validates non-null fields; the existence of IDs, session membership, and processing status are determined by the server.
  - `user.tool_confirmation` must include `tool_use_id`, and `result` can only be `allow` or `deny`.
  - `user.tool_result` is allowed only for `self_hosted` environments. Real execution checks `Config.Type` via `GetSession -> GetEnvironment`; `--dry-run` does not perform that read and does not prove the constraint is satisfied. Real execution and the server remain authoritative.
  - Raw payload can use snake_case; the PascalCase alias is normalized to snake_case before sending.

### Custom tool result

`user.custom_tool_result` is used to return the result of a specific custom tool call, and it must include the corresponding `custom_tool_use_id`:

```bash
arkcli agent session events send <session-id> \
  --type user.custom_tool_result \
  --custom-tool-use-id <custom-tool-use-id> \
  --content "tool output" \
  --format json
```

Equivalent structured event:

```json
{
  "type": "user.custom_tool_result",
  "custom_tool_use_id": "ctu-xxx",
  "content": [{"type": "text", "text": "tool output"}]
}
```

Sending without or with an empty `custom_tool_use_id` results in a validation error and no invocation of the data plane API. Do not use `user.tool_confirmation`'s `tool_use_id` to replace it; they belong to different event types.
