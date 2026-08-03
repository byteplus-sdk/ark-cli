# arkcli models custommodel upload

Import a custom model from TOS asynchronously.

> **Prerequisite: Activate BytePlus Torch Object Storage (TOS) and upload the model weights to a bucket before using the resulting `tos://<bucket>/<prefix>` URI with `--tos`.**
>
> - BytePlus TOS quick start: <https://docs.byteplus.com/en/docs/tos/docs-quick-start_1>
> - BytePlus TOS file upload guide: <https://docs.byteplus.com/en/docs/tos/docs-uploading-a-file>
>
> Follow one of these paths:
>
> - The user provides `tos://<bucket>/<prefix>`: confirm `--name`, `--base-model`, and `--tos`, then run `upload` after confirming the write operation.
> - The user has no TOS URI: **do not** run `upload`. Share the two links above and ask the user to enable TOS, create a bucket, upload the weights, and return with the URI.
>
> The bucket region must match the current BytePlus ModelArk region.

## Usage

```bash
arkcli models custommodel upload --name <name> --base-model <foundation-model-id> --tos tos://<bucket>/<prefix> [flags]
```

## Examples

```bash
# Minimal request with all three required parameters
arkcli models custommodel upload \
  --name my-finetune-v1 \
  --base-model <byteplus-foundation-model-id> \
  --tos tos://my-bucket/finetune/v1/

# Include a description and quantize during upload
arkcli models custommodel upload \
  --name my-finetune-v1-int8 \
  --base-model <byteplus-foundation-model-id> \
  --tos tos://my-bucket/finetune/v1/ \
  --quantization int8 \
  --description "lora-sft on customer support corpus, int8 quantized"

```

`upload` starts an asynchronous import that depends on TOS state and cannot
produce a reliable offline plan, so it does not register `--dry-run`. Restate
and confirm `name/base-model/tos/quantization` before real execution.

## Flags

| Parameter | Required | Type | Description |
|---|---|---|---|
| `--name` | Yes | string | Account-unique custom model name. It must begin with an ASCII letter; subsequent characters may be Chinese characters, ASCII letters, digits, underscores, or hyphens. |
| `--base-model` | Yes | string | BytePlus base model ID. Query it with `arkcli models search` under the target BytePlus profile; do not hard-code a model version that may become outdated. |
| `--tos` | Yes | string | TOS URI in `tos://<bucket>/<prefix>` form pointing to the weight directory. Create the bucket and upload the weights in the TOS console first. |
| `--quantization` | No | string | Quantization method to apply during upload; the model can also be quantized later with `quantize` |
| `--description` | No | string | Description, up to 300 characters |

## Output

Returns JSON containing the new `cm-xxxxx` and its initial status, usually `preparation`.

**Notes:**

- This is an **asynchronous** job. A successful response means only that the request was accepted, not that the weights are ready. Poll with `custommodel get <id>` until `status=ready` before deployment or quantization.
- The TOS bucket and current BytePlus ModelArk environment must be in the same region. Cross-region imports fail.
- If `--quantization` is provided during upload, the registry contains both the original and quantized `cm-xxxxx` models.
