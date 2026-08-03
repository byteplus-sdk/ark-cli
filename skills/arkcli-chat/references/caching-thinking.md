---
name: caching-thinking
description: arkcli +chat Caching / Thinking / ExpireAt usage reference, used with --store + chat get to implement verifiable multi-turn caching and thinking toggles.
---

# +chat Caching / Thinking / ExpireAt

These three flags let `+chat` actually control server-side caching, reasoning depth, and the persistence lifecycle, while echoing the state in the response so agents can verify it.

## When to use

- **Caching**: Use when the same system / tool definitions are repeated in a multi-turn conversation and you want to use the prompt cache to save cost; or when you want to disable caching to test clean behavior.
- **Thinking**: Use when the model supports reasoning but you do not want it to think in this turn (to save tokens), or when you want to force it to think.
- **ExpireAt**: Use when you want to add an expiration time to a response stored with `--store`, so it is not stored indefinitely.

## Flag quick reference

| flag | Type | Value / form |
|---|---|---|
| `--caching` | string | `enabled` / `disabled` (empty means use the server default) |
| `--cache-prefix` | bool | Used with `--caching`; enables prefix-cache (used by autotest's `prefix_cache_test`).**Mutually exclusive with `--max-output-tokens`** —— The server constraint is `caching.prefix is not supported when max_output_tokens is set`, and arkcli intercepts this combination on the client side. |
| `--thinking` | string | `auto` / `enabled` / `disabled` (empty means use the server default) |
| `--expire-at` | int (epoch sec) | Use with `--store`; if not passed, the server default GC policy is used. |

## Typical combinations

### Multi-turn conversation with caching enabled

```bash
# First turn: enable caching + store
RID=$(arkcli +chat "What color are strawberries?" --model ep-xxx \
  --caching enabled --store \
  --format json | jq -r .id)

# Second turn: enable caching as well, continue from RID
arkcli +chat "What about apples?" --model ep-xxx \
  --caching enabled --store \
  --previous-response-id "$RID"

# Verify cache hit: use chat get to retrieve the response; .caching.type should be enabled.
arkcli chat get "$RID" --format json | jq .caching
# → {"type":"enabled"}
```

### Disable thinking to shorten output

```bash
arkcli +chat "Introduce LLM in 30 characters" --model ep-xxx \
  --thinking disabled --max-output-tokens 100
# The response is returned directly without thinking phase.
```

### Persistence + expiration

```bash
TS=$(($(date +%s) + 3600))   # Expires in 1 hour
arkcli +chat "Remember that my name is Jim" --model ep-xxx \
  --store --expire-at "$TS"

# Verify
arkcli chat get "$RID" --format json | jq '{store, expire_at}'
# → {"store":true, "expire_at":1746676800}
```

## Output format

In non-streaming mode, both `+chat` and `chat get` return an extended `ResponsesResult`:

```json
{
  "id": "resp_...",
  "object": "response",
  "status": "completed",
  "created_at": 1746673200,
  "model": "...",
  "content": "...",
  "reasoning_content": "...",
  "usage": { ... },
  "store": true,
  "expire_at": 1746676800,
  "previous_response_id": "...",
  "caching": {"type": "enabled", "prefix": null},
  "thinking": {"type": "disabled"},
  "reasoning": {"effort": "medium"}
}
```

⚠️ In streaming mode (`--stream`), these echoed parameters are still **unavailable** in PR-2 — wait for PR-4 to add parsing for the `response.completed` event. During streaming, stdout is still `Thinking: ... Response: ...` plain text.

## Validation checklist (autotest mapping)

| Autotest test case | Unlocked |
|---|---|
| `Test_ResponseCreate_ExpireAtAndCaching` | ✅ |
| `Test_ResponseCreate_MaxOutputTokens` (includes Thinking disabled) | ✅ |
| `Test_ResponseCreateAndGet_PersistedFields` | ✅ (with PR-1 chat get) |
| 8 test cases of `cache/cache_test.go` | ✅ |
| `cache/prefix_cache_test.go` | ✅ (uses --cache-prefix) |
| `partial/partial_mode_test.go` | ✅ (Thinking + Caching combination) |
| `Test_Stream_ExpireAtAndCaching` | ⚠️ Input parameters can be passed, but output parameter echoes require PR-4 |
| 5 test cases of `cache/cache_stream_test.go` | ⚠️ Same as above |

## Common errors

| Symptom | Cause |
|---|---|
| `unsupported caching.type "X"` |The value is not `enabled` / `disabled`. |
| `unsupported thinking.type "X"` | The value is not `auto`/`enabled`/`disabled`. |
| `--cache-prefix` does not take effect. | `--caching enabled` was not also passed; the server rejects prefix alone by default. |
| `--expire-at` does not take effect. | `--store` was not passed; the server applies GC only to stored responses. |
| No echo is returned by `chat get` after setting it. | The response was not created with `--store` or has expired. |

## Use with reasoning-effort

`--reasoning-effort` (already available before PR-1) and `--thinking` are two independent dimensions:

- `--thinking disabled` directly disables the thinking phase, so `--reasoning-effort` no longer takes effect.
- `--thinking enabled` + `--reasoning-effort high` is required to actually trigger deep reasoning.
- autotest `Test_07_ThinkingReasoningCompatible` / `Test_08_ThinkingReasoningConflict` verify this combination.
