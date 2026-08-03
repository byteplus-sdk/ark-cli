# arkcli models custommodel available-quantizations

Query available quantization methods. Always run this command before `quantize`.

## Usage

```bash
arkcli models custommodel available-quantizations <id>
```

## Flags

| Parameter | Required | Type | Description |
|---|---|---|---|
| `<id>` | Yes | string | Custom model ID in `cm-xxxxx` form |

## Output

Returns JSON containing:

- `quantizations`: available quantization methods, such as `["W8A8"]`; the exact set varies by base model.
- `supported_inference_types_by_quantization`: a dictionary that predicts supported inference types before quantization. Each key is a quantization method, and its value lists the supported inference types, such as `{"W8A8":["token","model_unit"]}`.

**Notes:**

- This is a read-only command and can be run without confirmation.
- Do not infer support from the available methods of another base model.
- An empty list means the base model does not support quantization. Continue deploying the original model instead.
- `supported_inference_types_by_quantization` comes from ModelArk `QuantSupportedMethods` metadata. If the backend omits the field, the dictionary may be empty. After creation, use `custommodel get <new-id> --transform artifact_types` as the final source of truth for supported inference types.
