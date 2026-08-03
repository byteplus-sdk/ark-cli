---
name: tools
description: arkcli +chat tools usage reference, including the JSON file protocol for custom tools and tool_choice / max_tool_calls tuning.
---

# +chat Tools

PR-1 has the built-in `function` tool; `mcp` is left for PR-5b.

## When to use

- The user provides a function schema for the model to call → `function`
- The user asks "Help me look up meeting minutes" → The current PR does not support this; prompt the user to use `arkcli api arkruntime.create_responses --params '...'` (raw API) or wait for PR-5b.

## 4 new flags

| flag | Type | Purpose |
|---|---|---|
| `--tools <type>` | repeatable string | Simple syntactic sugar; supports only the single-parameter `{"type":"<type>"}` form. No tools are currently supported. |
| `--tools-file <path>` | string | Read a JSON file whose content is an array of tool objects in the full SDK form; function must use this method. |
| `--tool-choice <mode>` | string | Tool calling policy: `auto` / `required` / `none`. If omitted, the server default is used. |
| `--max-tool-calls <int>` | int | Maximum number of tool calls in a single session, to prevent uncontrollable behaviors.**Only takes effect for built-in tools**; when combined with a function tool (`--tools-file` or `--tools function:*`), arkcli intercepts it on the client side because the server reports `max_tool_calls only supported by build-in tools now`. |

`--tools` and `--tools-file` can be used together and are merged with "sugar first, file later".

## function (requires a JSON file)

`tools.json`:

```json
[
  {
    "type": "function",
    "name": "query_weather",
    "description": "Query the weather for a city and return JSON",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {"type": "string", "description": "City name, such as Beijing"}
      },
      "required": ["city"]
    }
  }
]
```

Call:

```bash
arkcli +chat "What's the weather like in Beijing today?" \
  --model ep-xxx \
  --tools-file tools.json \
  --tool-choice required
```

`function` object parameters (aligned with SDK `responses.ToolFunction`):

| Parameter | Required |Description|
|---|---|---|
| `type` | ✅ | Must be `"function"` |
| `name` | ✅ | Function name; the model uses it to identify the tool. |
| `description` | Recommended | Function description for the model; the clearer it is, the more accurately the model can trigger it. |
| `parameters` | Recommended | The parameters that describe the JSON Schema; write them directly as nested content, and arkcli internally serializes it into SDK Bytes. |
| `strict` | Optional bool | Strict mode: forces parameters to conform to the schema |

## Three tool_choice modes

| mode | Meaning |
|---|---|
| `auto` | Server default; lets the model decide whether to call a tool. |
| `required` | Forces at least one tool call. |
| `none` | Prevents the model from calling tools even if tools are passed. |

Complex function-specific tool_choice (`{"type":"function","name":"query_weather"}`) will be implemented in a later PR. For now, use `required` with a single tool to achieve the same effect.

## Output format

### Non-streaming (`+chat ...` without `--stream`)

`ResponsesResult` has an additional `function_calls` parameter:

```json
{
  "id": "resp_...",
  "model": "...",
  "content": "...",                  // The model's final text; it may be empty when a tool is triggered.
  "reasoning_content": "...",
  "usage": { ... },
  "function_calls": [
    {
      "type": "function_tool_call",
      "id": "call_...",
      "name": "query_weather",
      "arguments": "{\"city\":\"Beijing\"}",
      ...
    }
  ]
}
```

`function_calls[]` is passed through in the raw SDK form; different tool types have different parameters (`function_tool_call`, etc.). Use `jq '.function_calls[].type'` to distinguish them.

### Streaming (`--stream`)

Label lines such as `[Tool args] {"city":"Beijing"}` appear on stdout, inserted before or after `Response:` depending on the event order. Concatenating all chunks of `ChatStreamDelta.FunctionCallArgs` gives the complete args JSON.

## Error handling

| Symptom | Cause |
|---|---|
| `tool[N] type "X" not supported in this build` | A tool type supported only by PR-5b was used (`mcp`); use the raw API or wait for PR-5b. |
| `function tool requires 'name'` | The function is missing the name parameter. |
| `unsupported tool_choice "X"` | tool_choice is not auto/required/none. |
| Model replies but does not trigger the tool. | The tool description is not clear enough, tool_choice=none, or the question is unrelated to the tool. |
| Model repeatedly calls the same tool. | Add `--max-tool-calls` to limit it, or describe when to stop in the tool description. |

## Integration with chat get

After `--store` is added, even after streaming finishes, you can later use `chat get <id>` to get the complete `function_calls` list; `chat list-input-items <id>` can also show the complete tool call history (including result items such as `function_tool_call_output`).