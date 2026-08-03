---
name: text-format
description: arkcli +chat --text-format usage reference, makes the model output in the specified format (text / json_object / json_schema), with --text-schema for strong JSON Schema constraints.
---

# +chat Text Format

Make the model output in a structured format. The three modes cover the full spectrum from "free text" to "strict JSON Schema".

## When to use

- **text**: Default; free text.
- **json_object**: Makes the model output valid JSON without constraining shape; suitable for simple "return JSON" scenarios.
- **json_schema**: Strict JSON Schema; required when an agent / downstream program needs a stable shape (reduces parsing failures).

## Flag quick reference

| flag | Purpose |
|---|---|
| `--text-format` | text \| json_object \| json_schema |
| `--text-schema <path>` | JSON Schema file path; required in json_schema mode, ignored in other modes. |
| `--text-schema-name` | Schema name; displayed when the server echoes it; the default value is `arkcli_response`. |
| `--text-strict` | Strong constraint switch; only takes effect in json_schema mode. |

With `--text-strict`, the CLI also enforces the response contract locally:

1. The schema is compiled before the request. An invalid schema returns a validation error without a network call.
2. Only a completed response is accepted. An incomplete response, including token truncation, exits non-zero with `invalid_response`.
3. `content` must be a direct JSON value, without Markdown code fences or explanatory text.
4. The JSON value must validate against the same schema. A mismatch exits non-zero with `invalid_response`.

`--stream --text-strict` buffers events until the terminal event, validates the final content, and only then replays the events in order. Failed validation emits no partial JSON to stdout. Non-strict streaming and non-streaming behavior is unchanged.

## Typical usage

### json_object (simplest, makes the model output JSON)

```bash
arkcli +chat "What color are strawberries? Answer in JSON" --model ep-xxx \
  --text-format json_object
# {"color":"red"}
```

### json_schema (strongly constrained shape)

```bash
cat > schema.json <<'JSON'
{
  "type": "object",
  "properties": {
    "color": {"type": "string", "description": "Color name"},
    "hex":   {"type": "string", "pattern": "^#[0-9A-Fa-f]{6}$"}
  },
  "required": ["color", "hex"]
}
JSON

arkcli +chat "What color are strawberries? Provide the color name and hex" --model ep-xxx \
  --text-format json_schema --text-schema schema.json --text-strict
# {"color":"red","hex":"#FF3333"}
```

### Multi-turn continuation + json_schema

```bash
RID=$(arkcli +chat "What color are strawberries?" --model ep --store \
  --text-format json_schema --text-schema schema.json --text-strict \
  --format json | jq -r .id)

arkcli +chat "What about apples? Use the same format" --model ep --store \
  --previous-response-id "$RID" \
  --text-format json_schema --text-schema schema.json --text-strict
```

## Output format

In non-streaming mode (when `+chat ...` does not include `--stream`), `ResponsesResult` adds a `text_format` echo:

```json
{
  "id": "resp_...",
  "model": "...",
  "content": "{\"color\":\"red\",\"hex\":\"#FF3333\"}",
  "usage": { ... },
  "text_format": "json_schema"
}
```

`text_format` is the format actually applied by the server, and you can also retrieve it with `chat get $RID` (autotest jsonschema_test asserts this parameter when running chat get).

## Common errors

| Symptom | Cause |
|---|---|
| `--text-format=json_schema requires --text-schema <path>` | json_schema mode is missing --text-schema |
| `read --text-schema "X": no such file or directory` | Incorrect path or insufficient permissions |
| `unsupported text.format.type "yaml"` | Format value is not text / json_object / json_schema |
| `text.format.schema is required when type=json_schema` | Schema is omitted when using the raw API |
| `invalid --text-schema for --text-strict` | The schema is not valid or cannot be compiled; no request was sent. |
| `invalid_response` with `not completed` | The strict response was truncated; raise `--max-output-tokens` or request less output. |
| `invalid_response` with `not a direct JSON value` | The content contains a Markdown fence, explanation, or invalid JSON. |
| `invalid_response` with `did not match --text-schema` | The JSON does not match the schema; check the schema, model capability, and prompt. |

## Equivalent to the raw API

`+chat --text-format json_schema --text-schema f.json --text-strict --text-schema-name color` is equivalent to:

```bash
arkcli api arkruntime.create_responses --params '{
  "model":"ep-xxx",
  "input":"What color are strawberries?",
  "text":{
    "format":{
      "type":"json_schema",
      "schema":{...},
      "name":"color",
      "strict":true
    }
  }
}'
```

## Autotest unlock checklist

| Test case | Unlocked |
|---|---|
| `responseapi/jsonschema/TestJson1..3` | ✅ |
| `responseapi/jsonschema/TestJsonCache1..3` | ✅ |
| `responseapi/jsonschema/TestJsonSchema1..3` | ✅ |
| `responseapi/jsonschema/TestJsonSchemaCache1..3` | ✅ |
| `Test_ResponseCreate_TextFormat` (includes text + json_object) | ✅ |
| `Test_Stream_TextFormat` | ✅ (streaming input parameters can be passed; echo uses PR-4 --include-events) |
