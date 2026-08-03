---
name: arkcli-infer-endpoint
version: 1.3.0
description: "arkcli inference endpoint capability and lifecycle management: query model deployment forms with `arkcli infer endpoint capability get`, or get, list, start, stop, delete, and update existing Endpoints. Trigger for questions about shared service, batch, PTU, scale tier, fallback, supported inference methods, and whether a foundation/custom model can be deployed. Anti-trigger: creation intent routes to arkcli-deploy (`+deploy`) unless the user explicitly requests raw CRUD."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli infer endpoint --help"
---

# arkcli infer endpoint

**CRITICAL — Before starting, you MUST read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md).**

## Deployment capability routing

- Questions about a model's supported inference deployment forms belong to this Skill. This includes shared service, batch, PTU, scale tier, or fallback support.
- Run `arkcli infer endpoint capability get --model <model> --version <version> --format json` for a foundation model.
- Run `arkcli infer endpoint capability get --custom-model-id <cm-id> --format json` for a fine-tuned or imported custom model.
- Read [`references/arkcli-infer-endpoint-capability-get.md`](references/arkcli-infer-endpoint-capability-get.md) before executing either form.
- Do not route deployment-form questions to `arkcli-models`, `models get`, or Raw API Explorer. `arkcli-models` describes catalog/model capabilities; `infer endpoint capability get` is authoritative for deployment methods.

## Usage principles

- Prefer `arkcli infer endpoint ...` for inference endpoint requests.
- Although these commands are standard CLI commands, their implementation entry point still comes from `shortcuts/inferendpoint/`.
- Only fall back to [`../arkcli-api-explorer/SKILL.md`](../arkcli-api-explorer/SKILL.md) when product commands cannot cover the request.
- After `infer endpoint create` succeeds, it returns the Endpoint `Id`.
- If you already obtained an `Id` through `infer endpoint create`, do not call `+deploy` again to attempt a "second deployment"; `+deploy` itself is the workflow that creates an Endpoint.
- `infer endpoint create --billing-method` currently supports only `token`. This flag is optional. When `token` is passed explicitly, the command first checks whether the model supports token-based inference, while the create request itself keeps the default behavior.
- **Voice models cannot be used to create Endpoints**: For TTS / ASR / dubbing / podcast / voice / real-time voice interaction, do not use `infer endpoint create` or raw CRUD as a workaround. Clearly state that arkcli currently does not support Endpoint creation for these models.


## Semantics of "my endpoints"

When the user says **"my inference endpoints / endpoints I created / how many endpoints I have / list mine"**, you MUST add `--mine --page-all`:

```bash
arkcli infer endpoint list --mine --page-all --page-size 100 --format json
```

The server filters by the `sys:ark:createdBy` tag and returns only endpoints created by the current SSO sub-user.
This requires an SSO sub-user login. Root accounts / AK-SK logins fail directly, and you should guide the user to log in again.
For detailed behavior, see [`references/arkcli-infer-endpoint-list.md`](references/arkcli-infer-endpoint-list.md).

## Boundary between `infer endpoint create` and `+deploy`

`infer endpoint create` is **raw CRUD**. It is suitable for scripting / CI / scenarios that need predictable behavior without workflow guardrails. Note that its flag set is actually a **subset** of `+deploy` (`+deploy` has the superset of parameters). The difference is whether workflow guardrails are present, not which command has more parameters. Its guardrails differ from those of `+deploy` as follows:

| Guardrail | `+deploy` | `infer endpoint create` |
|------|-----------|------------------------|
| Real-name prerequisite check ([realname-gate.md](../arkcli-auth/references/realname-gate.md)) | ✅ | ✅ (the foundation model path also triggers `EnsureModelAvailable`, including real-name blocking) |
| Custom model reuse check (avoids duplicate billing) | ✅ | ❌ |
| Default profile resource sync after deployment (`--set-default`) | ✅ | ❌ |

**Routing decision**: As long as the user's intent is to "create / add / set up an endpoint / access point" or "deploy / launch a model" (regardless of whether the wording is create or deploy), always route to `+deploy` ([arkcli-deploy](../arkcli-deploy/SKILL.md)). Use `infer endpoint create` **only** when the user explicitly needs to bypass workflow guardrails, run scripting / CI, or needs predictable raw CRUD behavior without guardrails.

**Exception**: If the target is a voice model (TTS / ASR / podcast / dubbing / voice / real-time voice interaction), do not enter this skill just because the user said "create endpoint". arkcli currently only supports discovering voice models through `models search`; it does not support Endpoint creation for voice models.

## Command overview

| Command | Description |
|------|------|
| `arkcli infer endpoint capability get` | Query supported deployment forms for a foundation model or custom model. |
| `arkcli infer endpoint create` | Create an inference endpoint (raw CRUD, no deployment workflow guardrails). For the default intent of creating a new endpoint, use `+deploy`; this command is only for scripting / raw CRUD scenarios without guardrails. |
| `arkcli infer endpoint get <endpoint-id>` | Get inference endpoint details. |
| `arkcli infer endpoint list [--mine]` | List inference endpoints. When the user says **"mine / my own / created by me / how many I have"**, you MUST add `--mine` (SSO sub-user filtering). |
| `arkcli infer endpoint start <endpoint-id>` | Start an inference endpoint. |
| `arkcli infer endpoint stop <endpoint-id>` | Stop an inference endpoint. |
| `arkcli infer endpoint delete <endpoint-id>` | Delete an inference endpoint (**irreversible**; requires secondary confirmation. In non-interactive environments, `ARKCLI_ALLOW_HEADLESS_DELETE=1` is required; `--yes` does not authorize deletion). |
| `arkcli infer endpoint update <endpoint-id>` | Update an inference endpoint (name / description / rate limits). |

## References

- [`references/arkcli-infer-endpoint-capability-get.md`](references/arkcli-infer-endpoint-capability-get.md)
- [`references/arkcli-infer-endpoint-create.md`](references/arkcli-infer-endpoint-create.md)
- [`references/arkcli-infer-endpoint-get.md`](references/arkcli-infer-endpoint-get.md)
- [`references/arkcli-infer-endpoint-list.md`](references/arkcli-infer-endpoint-list.md)
- [`references/arkcli-infer-endpoint-start.md`](references/arkcli-infer-endpoint-start.md)
- [`references/arkcli-infer-endpoint-stop.md`](references/arkcli-infer-endpoint-stop.md)
- [`references/arkcli-infer-endpoint-delete.md`](references/arkcli-infer-endpoint-delete.md)
- [`references/arkcli-infer-endpoint-update.md`](references/arkcli-infer-endpoint-update.md)
