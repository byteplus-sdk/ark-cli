# infer endpoint capability get

Query the deployment methods that the BytePlus control plane reports for a foundation model version or a custom model.

## Routing

Use this command when the user asks whether a model supports any of these deployment forms:

- shared service or online endpoint
- batch inference
- provisioned throughput unit (PTU)
- scale tier
- endpoint fallback

These are deployment capabilities, not marketplace metadata. Do not substitute `arkcli models get` or `arkcli models search`.

## Commands

```bash
# Foundation model
arkcli infer endpoint capability get --model <model> --version <version> --format json

# Fine-tuned or imported custom model
arkcli infer endpoint capability get --custom-model-id <cm-id> --format json
```

`--model` and `--custom-model-id` are mutually exclusive. Supply the exact model version when the user names or needs a version-specific foundation-model result.

## Output contract

The JSON response contains:

- `model_reference`: the foundation-model/version pair or custom model ID sent to the control plane.
- `supported_methods`: the names whose capability flags are true; use this as the concise Agent answer.
- `capabilities`: the complete boolean map, including false values.

Typical capability keys include `share_service`, `batch_inference`, `provisioned_throughput_unit`, `scale_tier`, and `endpoint_fallback`.

## Interpretation

- Report only the methods returned as supported; do not infer support from model naming, modality, or another product's catalog.
- An empty `supported_methods` result means no method was reported as supported for that exact reference. It is not permission to retry through `models get`.
- Authentication, permission, timeout, or transport errors must be returned as errors. Do not replace them with guessed capabilities.
- A capability query is read-only. It does not create, update, start, or bill an Endpoint.

## Follow-up

- If the user only asked what is supported, stop after reporting the result.
- If the user then asks to deploy, route to [`arkcli-deploy`](../../arkcli-deploy/SKILL.md) for the guarded workflow.
- Use raw `infer endpoint create` only when the user explicitly requests predictable CRUD behavior without the `+deploy` workflow guardrails.
