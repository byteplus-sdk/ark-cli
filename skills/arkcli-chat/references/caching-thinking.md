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

# Check cache configuration, not a cache hit: retrieve the response with chat get.
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

For machine-readable streaming evidence, use `--stream --include-events` and inspect the
successful terminal event's actual response fields. Plain `--stream` is human-readable,
not the flat JSON above. Missing echoes remain unknown; do not add another inference call
just to obtain them. See [stream-events.md](stream-events.md).

## Validation checklist (autotest mapping)

| Autotest test case | Unlocked |
|---|---|
| `Test_ResponseCreate_ExpireAtAndCaching` | ✅ |
| `Test_ResponseCreate_MaxOutputTokens` (includes Thinking disabled) | ✅ |
| `Test_ResponseCreateAndGet_PersistedFields` | ✅ (with PR-1 chat get) |
| 8 test cases of `cache/cache_test.go` | ✅ |
| `cache/prefix_cache_test.go` | ✅ (uses --cache-prefix) |
| `partial/partial_mode_test.go` | ✅ (Thinking + Caching combination) |
| `Test_Stream_ExpireAtAndCaching` | Inspect actual terminal events with --include-events |
| 5 test cases of `cache/cache_stream_test.go` | Inspect actual terminal events; configuration alone does not prove a cache hit |

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

- Use `--thinking disabled` when the user asks to disable thinking. `--reasoning-effort minimal` controls effort and is not a replacement for that switch.
- These options map to Responses request fields `thinking.type` and `reasoning.effort` respectively. Preserve explicit values in SDK/HTTP examples; do not put effort inside thinking or automatically add high effort.
- Accepted combinations and effective behavior depend on the target model and server. Preserve actual errors and echoes rather than promising universal support.
- `caching.type=enabled` is configuration, not proof of a cache hit. Use actual usage cache-hit counters when available; missing counters mean unknown savings.
- Latency comparisons need the exact model version, inputs, settings, cache state, and measured time. Do not claim a universal disabled/minimal/high ordering or add paid requests merely to test a guess.
