# arkcli-infer-endpoint evaluation cases

## Routing goals

- Deployment-form questions must route to `arkcli-infer-endpoint`, even when the prompt contains the generic word "capability".
- Foundation models use `--model` plus `--version`; custom models use `--custom-model-id`.
- Marketplace metadata commands must not substitute for the authoritative deployment-method query.

## Regression cases

| case | prompt | expected |
|---|---|---|
| `endpoint-capability-foundation` | Which deployment forms does `dola-seed-2-1-turbo` version `260628` support, such as shared service, batch, PTU, scale tier, or fallback? | Load `arkcli-infer-endpoint` and run `arkcli infer endpoint capability get --model dola-seed-2-1-turbo --version 260628 --format json`. |
| `endpoint-capability-custom` | Can custom model `cm-123` use PTU or shared service? | Run `arkcli infer endpoint capability get --custom-model-id cm-123 --format json`. |
| `endpoint-capability-read-only` | Check whether this model is deployable, but do not create anything. | Run only the capability query; do not run `+deploy` or `infer endpoint create`. |
| `endpoint-capability-no-models-fallback` | Tell me the supported inference deployment methods. | Do not use `models get`, `models search`, or Raw API Explorer as a replacement. |

## Key scoring points

- Must route to `arkcli-infer-endpoint`.
- Must execute `arkcli infer endpoint capability get`.
- Must read `supported_methods` and the full `capabilities` map.
- Must not infer deployment support from model name, modality, or another product.
- Must not mutate an Endpoint during a capability-only request.
