# pricing models

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

List the settlement unit price catalog for ARK foundation models. `Price` is the **final unit price (including discounts)** calculated by the backend according to the current account's contract / promotions / plans; `OriginalPrice` is the published list price.

**Voice models are unsupported**: For TTS / ASR / dubbing / podcast / voice / real-time voice interaction, do not query prices with `pricing models --model` or `--modality Audio`. Clearly state that voice models are unsupported.

## Commands

```bash
# List all foundation models (default PageSize=50, PageNumber=1)
arkcli pricing models

# Query one foundation model
arkcli pricing models --model deepseek-v4-pro

# Query one custom model (a fine-tuned cm-* model)
# The CLI first calls GetCustomModel to resolve the base model, then performs the standard catalog query;
# the top-level response includes an additional resolved_custom_model field
arkcli pricing models --model cm-20260611124416-77wrq

# Filter by modality (backend FoundationModelDomain enum)
arkcli pricing models --modality LLM
arkcli pricing models --modality ComputerVision
arkcli pricing models --modality Embedding

# Pagination
arkcli pricing models --page-size 20 --page-number 2

# Specify the pricing region (defaults to the current profile region)
arkcli pricing models --region ap-southeast-1
```

## Parameters

### Optional filters

| Parameter | Type | Description |
|------|------|------|
| `--model` | string | One foundation model name (such as `dola-seed-2-1-turbo` / `deepseek-v4-pro`) or custom model ID (`cm-*`). The CLI automatically resolves the base model for `cm-*` and adds `resolved_custom_model` to the response |
| `--modality` | string | Backend `FoundationModelDomain` enum; **aliases are not accepted**: `LLM` / `ComputerVision` / `Audio` / `Embedding` / `Router` |
| `--region` | string | Pricing region; defaults to the current profile region |
| `--page-number` | int | Page number, 1-based, default 1 |
| `--page-size` | int | Number of entries per page, default 50 |

### Using `--modality`

The backend `FoundationModelDomain` field **does not distinguish** image / video / 3D; they all belong to `ComputerVision`.

| User semantics | Value to pass | Note |
|---|---|---|
| Text / large language model | `LLM` | |
| Image generation / image | `ComputerVision` | Includes image / video / 3D |
| Video generation | `ComputerVision` | Same as above; requires secondary client-side filtering |
| Audio / voice | Do not query | `Audio` is a backend enum, but this arkcli skill does not support voice model pricing queries |
| Vector | `Embedding` | |
| Intelligent routing | `Router` | |

⚠️ Do not pass natural-language aliases such as `text` / `video` / `image`. arkcli passes the value unchanged to the backend, while the backend recognizes only five enum values, resulting in an empty result or an error.

## Return value

JSON structure (selected key fields):

```json
{
  "page_number": 1,
  "page_size": 50,
  "total_count": 75,
  "items": [
    {
      "FoundationModelName": "deepseek-v4-pro",
      "DisplayName": "DeepSeek-V4-pro",
      "VendorName": "DeepSeek",
      "State": "Available",
      "ChargeItems": [
        { "Type": "InferencePrompt",     "Price": 0.012, "OriginalPrice": 0.012, "UnitCode": "k tokens" },
        { "Type": "InferenceCompletion", "Price": 0.024, "OriginalPrice": 0.024, "UnitCode": "k tokens" },
        { "Type": "ContextSessionHit",   "Price": 0.001, "UnitCode": "k tokens" },
        { "Type": "ContextSessionStorage","Price": 0.000017, "UnitCode": "k tokens/hour" },
        { "Type": "BatchInferencePrompt","Price": 0.006, "UnitCode": "k tokens" },
        { "Type": "BatchInferenceCompletion","Price": 0.012, "UnitCode": "k tokens" }
      ],
      "InferenceFreeUsage": { "Total": 500000, "Consumed": 0 },
      "ResourcePackItems": [
        { "Type": "FreeInference", "Total": 500000, "Consumed": 500000 }
      ],
      "SubServices": [{ "SubService": "base", "Status": "Available" }],
      "IsOverdue": false
    }
  ]
}
```

## Key field definitions

| Field | Meaning |
|---|---|
| `Price` | **Discounted unit price for the current account** (the backend has applied contracts/promotions/plans); the Agent should display this value directly |
| `OriginalPrice` | Published list price. `Price < OriginalPrice` indicates a discount; the difference/ratio can describe the discount amount |
| `UnitCode` | Billing unit, commonly `k tokens`; storage uses `k tokens/hour` |
| `Type` | ChargeItem type (see the table below). When reporting inference prices, normally show only `InferencePrompt` + `InferenceCompletion` |
| `State` | Whether the model is available to the current account (`Available` / `Unavailable`) |
| `InferenceFreeUsage` | Free inference quota: `Total` total amount, `Consumed` consumed amount |
| `ResourcePackItems` | Resource pack consumption; type `FreeInference` is the free inference resource pack |
| `SubServices` | Sub-service activation state (`base` / `context-cache`), related to `--moderation` and caching behavior in [arkcli-deploy](../../arkcli-deploy/SKILL.md) |
| `IsOverdue` | true indicates overdue payment, which affects inference call success |
| `MultiChargeItems` | Tiered pricing (by token-length range), available for a few models |
| `resolved_custom_model` | Appears **only when `--model` is passed a cm-*** (top-level field, not inside items); see the next section |

## Custom model (cm-*) reverse lookup

The backend `ListModelChargeItems` does not index custom model entries. Rates for fine-tuned cm-* models are embedded in the base model's ChargeItems with `Finetune*`-prefixed types. The CLI performs the reverse lookup for the user:

**Trigger condition**: `--model` receives any string beginning with `cm-`.

**Execution flow**:
1. The CLI first calls `custom_model.get_custom_model { Id: "cm-xxx" }`.
2. It obtains `FoundationModel.Name` / `FoundationModel.ModelVersion` / `CustomizationType` from the result.
3. It runs the standard `ListModelChargeItems` query using the base name.
4. Response `Items` contains all ChargeItems for the base model (**not trimmed**), and the top level includes an additional `resolved_custom_model`.

**Additional response field**:

```json
{
  "resolved_custom_model": {
    "id": "cm-20260611124416-77wrq",
    "customization_type": "FinetuneSft",
    "foundation_model_name": "dola-seed-2-1-turbo",
    "foundation_model_version": "260628"
  },
  "page_number": 1,
  "page_size": 50,
  "total_count": 1,
  "items": [ /* all ChargeItems for the base model; the Agent selects Finetune* entries */ ]
}
```

**ChargeItem selection for the Agent (by `customization_type`)**:

| customization_type | Invocation rate (select these two when asked about cm-* inference price) | Training rate (select this when asked about the cost to train it) |
|---|---|---|
| `FinetuneSft` (full SFT) | `FinetuneInferencePrompt` + `FinetuneInferenceCompletion` | `Finetune` |
| `FinetuneLoRA` / `DPOLoRA` / `GRPOLoRA` / `OPDLoRA` | `FinetuneInferencePrompt` + `FinetuneInferenceCompletion` | `LoraFinetune` |
| `DPO` / `GRPO` / `PPO` / `OPD` (full RL) | `FinetuneInferencePrompt` + `FinetuneInferenceCompletion` | `Finetune` |
| `Pretrain` | `FinetuneInferencePrompt` + `FinetuneInferenceCompletion` | `Finetune` |

> Invocation rates for cm-* models fine-tuned for vision/video use `FinetuneT2ICompletion` / `FinetuneI2ICompletion` / `FinetuneT2VCompletion` / `FinetuneI2VCompletion` / `FinetuneTo3DCompletion`, and others. See the complete ChargeItem.Type enum mapping below.

**Reverse-lookup errors**:
- `cm-xxx` does not exist → `GetCustomModel "cm-xxx": ... The specified CustomModel cm-xxx is not found.`
- No permission → passes through `AccessDenied`; route to [arkcli-auth](../../arkcli-auth/SKILL.md).

## Complete ChargeItem.Type enum mapping

The base model's `ChargeItems` array is **comprehensive**. All types below may appear under the same base model node. The Agent should select the relevant entry based on the user's question.

### Inference types (direct invocation of the base model)

| Type | Meaning |
|---|---|
| `Inference` | Average overall inference price (provided by some models) |
| `InferencePrompt` | Inference input price |
| `InferenceCompletion` | Inference output price |
| `BatchInferencePrompt` | Batch inference input price |
| `BatchInferenceCompletion` | Batch inference output price |
| `BatchInferenceCacheHit` | Batch inference prompt cache-hit price |
| `ContextSessionHit` | Context Cache hit price |
| `ContextSessionStorage` | Context Cache storage price (unit: `k tokens/hour`) |
| `VisionPrompt` / `BatchVisionPrompt` | Multimodal vision input price |

### Fine-tuning training types (training rates when using a base model for fine-tuning)

| Type | Meaning |
|---|---|
| `Finetune` | Full fine-tuning training price |
| `LoraFinetune` | LoRA fine-tuning training price |

### Fine-tuned inference types (inference price for invoking a fine-tuned custom model)

| Type | Meaning |
|---|---|
| `FinetuneInferencePrompt` | Fine-tuned model inference input price |
| `FinetuneInferenceCompletion` | Fine-tuned model inference output price |
| `FinetuneI2ICompletion` | Fine-tuned model image-to-image output price |
| `FinetuneI2VCompletion` | Fine-tuned model image-to-video output price |
| `FinetuneT2ICompletion` | Fine-tuned model text-to-image output price |
| `FinetuneT2VCompletion` | Fine-tuned model text-to-video output price |
| `FinetuneTo3DCompletion` | Fine-tuned model 3D generation output price |

### Vision/multimodal generation types (output price when a base model performs generation)

| Type | Meaning |
|---|---|
| `T2ICompletion` / `T2ITokenCompletion` | Text-to-image |
| `I2ICompletion` / `I2ITokenCompletion` | Image-to-image |
| `T2VCompletion` / `I2VCompletion` | Text-to-video / image-to-video |
| `V2VCompletion` / `V2V1080Completion` / `V2V4KCompletion` | Video-to-video (standard / 1080p / 4K) |
| `NV2VCompletion` / `NV2V1080Completion` / `NV2V4KCompletion` | Silent video-to-video |
| `FLF2VCompletion` | First-and-last-frame-to-video |
| `To3DCompletion` / `BatchTo3DCompletion` | 3D generation |
| `ToVCompletion` / `ToVSilentCompletion` / `BatchToVCompletion` / `BatchToVSilentCompletion` | General video generation (including silent video) |
| `BatchV2VCompletion` / `BatchV2V1080Completion` / `BatchV2V4KCompletion` | Batch video-to-video |

### Audio types

| Type | Meaning |
|---|---|
| `FastAudioPrompt` / `FastAudioCacheHit` | Low-latency audio inference (low-latency inference is not currently supported) |

> **Important**: A user question usually concerns only 1–2 types. Common mappings:
> - "How much is inference?" → `InferencePrompt` + `InferenceCompletion`
> - "How much does fine-tuning cost?" → `Finetune` or `LoraFinetune` (multiplied by `tokens × epoch`)
> - "How much does it cost to invoke a fine-tuned model?" → `FinetuneInferencePrompt` + `FinetuneInferenceCompletion`
> - "How much is image-to-image?" → `T2ICompletion` / `I2ICompletion` (base model) or `FinetuneT2ICompletion` / `FinetuneI2ICompletion` (fine-tuned model)

## Common usage

**Report a model's input/output price to the user**

```bash
arkcli pricing models --model dola-seed-2-1-turbo \
  --transform 'items.0.ChargeItems.#(Type=="InferencePrompt").Price'
```

Alternatively, omit transform to retrieve all entries, and let the Agent select `InferencePrompt` / `InferenceCompletion`.

**Determine whether the account has a discount**

```bash
arkcli pricing models --model X
# Compare Price and OriginalPrice; Price < OriginalPrice means there is a discount
```

**List and compare prices for all models in a modality**

```bash
arkcli pricing models --modality LLM --page-size 100
# The Agent sorts and summarizes the result by vendor / FoundationModelName
```

## Common errors

| Error | Cause | Handling |
|------|------|---------|
| Empty items / total_count=0 | `--modality` was passed an alias such as `text` / `video` instead of a backend enum (this path applies only when `--model` is omitted) | Translate it to `LLM` / `ComputerVision`, and others, before passing it |
| `foundation model "X" not found in pricing list` (exit=2) | Model name X was not found in the pricing list | Check the spelling, or run `arkcli pricing models` without `--model` to retrieve all supported names |
| Model State=Unavailable | The current account has not activated it | Route to `arkcli +deploy` (automatic activation) or `arkcli models activate` |
| `IsOverdue=true` | The account has overdue payments | Does not affect pricing queries, but affects inference calls |
| `GetCustomModel "cm-xxx": ... is not found` | The cm-prefixed ID does not exist or does not belong to the current account | Check the ID spelling, or first run `arkcli api custom_model.list_custom_models` to retrieve the list |

## References

- [arkcli-pricing](../SKILL.md) — Pricing skill overview
- [arkcli-shared](../../arkcli-shared/SKILL.md) — Authentication and global parameters
- [arkcli-usage](../../arkcli-usage/SKILL.md) — View actual consumption (tokens / requests already consumed)
