# arkcli models custommodel quantize

Quantize a custom model asynchronously.

## Usage

```bash
arkcli models custommodel quantize <id> --quantization <mode> [flags]
```

## Examples

```bash
# Standard usage: first run available-quantizations to validate the mode and inspect the inference types it is expected to support
arkcli models custommodel quantize cm-xxxxx --quantization int8

# Client preview: inspect the request without creating a quantized model
arkcli models custommodel quantize cm-xxxxx \
  --quantization int8 \
  --dry-run

# Include a description
arkcli models custommodel quantize cm-xxxxx \
  --quantization int8 \
  --description "int8 quantized for low-latency inference"
```

## Flags

| Parameter | Required | Type | Description |
|---|---|---|---|
| `<id>` | Yes | string | Source custom model ID; the model must have `status=ready` |
| `--quantization` | Yes | string | Quantization method. **Select a value returned by `available-quantizations <id>`.** |
| `--description` | No | string | Description for the quantized model, up to 300 characters |
| `--dry-run` | No | bool | Command-local Client Preview that emits `preview.v1` without a network request |

## Output

Returns JSON containing a newly created `cm-yyyyy`, which differs from the source ID, and its initial status.

**Notes:**

- `--dry-run` never calls `CreateQuantizedCustomModel`, and `preview.v1.effects.network` must be `blocked`. The locally built `steps[].payload` must not contain a backend `DryRun` field; this is not server-side validation.
- This is an **asynchronous** job. A successful response means only that the request was accepted. Poll the new ID with `custommodel get <new-id>` until it reaches `ready`.
- The quantized result is an **independent new custom model**. The source remains unchanged, so both `cm-xxxxx` resources exist in the account.
- Quantization can materially affect inference performance and accuracy. Before production use, consider deploying the source and quantized models to separate endpoints for comparison.
- To determine whether a quantization method is expected to support pay-as-you-go by token usage or Model Unit inference, inspect `supported_inference_types_by_quantization` from `available-quantizations` before creation.

## Resource relationships

The quantization flow contains three resource types. Never mix their IDs.

| Resource | ID form | Description |
|---|---|---|
| Source custom model | `cm-xxxxx` | Input to `quantize`; remains unchanged |
| Quantized model | New `cm-yyyyy` | New model returned by `quantize`; continue polling it with `get` until `ready` |
| Inference endpoint | `ep-xxxxx` | Created through `+deploy` from a ready source or quantized model; `+chat` and `+gen` should use an endpoint or invocable model ID |

Before creation, use `supported_inference_types_by_quantization` from `available-quantizations` to predict supported inference types. After creation, use `supported_inference_types` from `custommodel get <new-id> --transform artifact_types` for final confirmation.
