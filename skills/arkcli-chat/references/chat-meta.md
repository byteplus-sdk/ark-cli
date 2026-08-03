---
name: chat-meta
description: Reference for the three standard CRUD commands arkcli chat get / delete / list-input-items, used with the +chat workflow to manage stored conversations.
---

# chat get / delete / list-input-items

This is the "reconciliation page" for `+chat`: `+chat` starts and persists conversations; these three commands query, delete, and view input history afterward.

## When to use

- `chat get`: You already have a response_id and want to retrieve the full conversation content (including reasoning, function calls, and usage).
- `chat delete`: Delete the response record left by a previous `--store`.
- `chat list-input-items`: View the input list for a response (useful for tracing history after multi-turn continuation).

## Prerequisites

- The response operated on by these three commands **must have been persisted by `+chat --store`**. A response without `--store` is not retained on the server; `chat get` returns `InvalidParameter.PreviousResponseNotFound`.
- Authentication is the same as `+chat`; see `../arkcli-auth/SKILL.md`.

## Command quick reference

```bash
# 1. Create a conversation and persist it, then get the response ID.
RID=$(arkcli +chat "What color are strawberries?" --model ep-xxx --store --format json | jq -r .id)

# 2. Query.
arkcli chat get "$RID"

# 3. Have the server expand the base64 of input_image when querying (useful for debugging image input in practice).
arkcli chat get "$RID" --include image_url

# 4. List input items (latest 5 in reverse order).
arkcli chat list-input-items "$RID" --order desc --limit 5

# 5. Pagination (continue fetching after getting first_id / last_id).
arkcli chat list-input-items "$RID" --after <last_id_from_previous_page> --limit 20

# 6. Delete.
arkcli chat delete "$RID"
```

## Flag descriptions

### chat get

| flag |Description|
|---|---|
| `--include` | Have the server expand extension parameters in the response; can be repeated. Optional values (validated locally by the CLI): `image_url` / `audio_url` / `encrypted_content`.**Warning**: The SDK and production backend currently have a skew in the include enum strings. Some accounts / access points return `Invalid value for 'include'` for all three values. If you encounter this error, just remove `--include` (the response body content will still be returned normally). |

### chat list-input-items

| flag |Description|
|---|---|
| `--after <item_id>` | Cursor pagination: list content after this item. |
| `--before <item_id>` | Cursor pagination: list content before this item. |
| `--limit <int>` | Page size, 1–100; the server default is 20. |
| `--order asc\|desc` | Sort direction; the server default is desc. |
| `--include` | Same as `chat get`. |

### chat delete

No extra flags.

## Output format

- `chat get` reuses the `ResponsesResult` structure of `+chat`: `{id, model, content, reasoning_content, usage, function_calls?}`.
- `chat list-input-items` returns `{object, data:[…], first_id, last_id, has_more}`. `data[]` is the raw SDK-format InputItem JSON (16-way oneof, with no arkcli-side mirror). Use `jq '.data[].type'` to see which type each item is, such as `input_message` / `function_tool_call` / `function_web_search` / `reasoning` / etc.
- `chat delete` returns `{id, deleted:true}`. The server has no response body; this is synthesized by arkcli.

## Common errors

| Symptom | Cause |
|---|---|
| `InvalidParameter.PreviousResponseNotFound` | response_id is misspelled, or `--store` was not used at creation time. |
| `chat get/delete/list-input-items` reports `missing response id`. | The positional arg was not provided. |
| `tool/function call` is not shown in `data[]`. | `--tools` was not passed when creating it; or the model did not trigger a tool, see `references/tools.md`. |