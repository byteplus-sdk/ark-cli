---
name: arkcli-pricing
version: 1.0.0
description: "Queries BytePlus ModelArk foundation model settlement unit prices (including current-account discounts) and CodingPlan subscription prices. Price is the final unit price after account contract / promotion / plan discounts; OriginalPrice is the published list price. Invoke for model prices, pricing, unit prices, Coding Plan prices, plan prices, discounted prices, token billing, cross-modality price comparisons, or model free quotas. Anti-trigger: TTS/ASR/voice model pricing is unsupported; do not use Audio pricing and clearly state that it is unsupported."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli pricing --help"
---

# arkcli pricing

**CRITICAL — Before starting, you MUST use the Read tool to read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md), which contains authentication gates, configuration troubleshooting, and shared safety rules.**

## Applicable scenarios

- Query settlement unit prices for foundation models (input / output / batch inference / context cache and other ChargeItems).
- View free quotas (`InferenceFreeUsage`, `ResourcePackItems`) and overdue status (`IsOverdue`).
- View sub-service activation states (`SubServices`: `base` / `context-cache`).
- Filter the pricing catalog by modality.
- Query **CodingPlan subscription prices** (personal + enterprise, four tiers in total).

## Business positioning

- **The two pricing categories are managed separately**: **model pay-as-you-go pricing** (`pricing models`) and **plan subscription pricing** (`pricing plans`) are completely independent commands. See "Models vs plans" below for the decision.
- This covers only catalog pricing for **foundation models**. However, a base model's ChargeItems are **comprehensive**: they include foundation model inference prices, training rates when the model is used for fine-tuning (`Finetune` / `LoraFinetune`), and inference prices for custom models produced by fine-tuning (`FinetuneInference*` / `FinetuneI2I*`, and others). Therefore, querying a base model once returns all related rates.
- The **backend API** does not return separate entries for custom models (fine-tuned `cm-*` models), because their rates are embedded in the base model's ChargeItems array. However, **arkcli provides reverse-lookup convenience**: `pricing models --model cm-xxx` automatically calls GetCustomModel to obtain the base name and then queries pricing. The top-level response also includes `resolved_custom_model` (`customization_type` + `foundation_model_name` / `foundation_model_version`). The Agent can use it to select the corresponding ChargeItem.Type without manually performing a two-step lookup.
- Endpoint-level actual consumption is not covered → use [arkcli-usage](../arkcli-usage/SKILL.md).
- **Rate limits / token limits / context windows are not included**. Those belong to `arkcli-models`; route to [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md). Pricing handles only "money".
- does not offer Agent Plan, do not run `arkcli pricing plans` and do not present Coding Plan prices as Agent Plan prices.
- **Voice model pricing is not covered**: TTS / ASR / dubbing / reading / podcast / voice / real-time voice interaction models are currently unsupported in arkcli. Do not use `pricing models --model` or `--modality Audio` to answer their prices.
- **Price = the final unit price including the current account's discount** (already calculated by the backend based on contracts/promotions/plans). No client-side discount calculation is needed.
- **OriginalPrice = the published list price**. `Price < OriginalPrice` indicates that the account has a discount.

## Commands

| Command | Description |
|------|------|
| [`pricing models`](references/arkcli-pricing-models.md) | List the foundation model settlement pricing catalog (pay-as-you-go, token-based billing). |
| [`pricing plans`](references/arkcli-pricing-plans.md) | List subscription plan prices (CodingPlan personal + enterprise). |

## Models vs plans: which command to use

**Is the user asking for "a unit price calculated by token" or "a monthly plan price"? This is the key distinction.**

| User wording | Use | Reason |
|---|---|---|
| "How much does DeepSeek-V4 cost?" / "How much is dola-seed?" / "GPT-X price" | `pricing models` | Model name → token-based billing |
| "How much does image-to-image cost per image?" | `pricing models` | Per-call billing in ChargeItems |
| "How much is audio recognition per minute?" / "How much is TTS?" / "Voice model pricing" | `arkcli-models` | arkcli currently supports only voice model marketplace discovery, not pricing queries |
| "How much does fine-tuning cost?" / "How much does it cost to fine-tune a model?" | `pricing models` | Fine-tuning rates are embedded in the base model's ChargeItems (see `Finetune` / `LoraFinetune` types) |
| "How much does my cm-xxx cost to invoke?" / "How much is this fine-tuned model?" | `pricing models --model cm-xxx` | The CLI automatically resolves the base model + exposes `resolved_custom_model.customization_type`; select the ChargeItem according to the table below |
| "How much is Coding Plan?" / "lite / pro plan price" | `pricing plans` | Plan name → monthly subscription |
| "Coding Plan enterprise price" / "How much is the team edition?" | `pricing plans --edition enterprise` | Same as above |

Ambiguity fallback: if the user asks "How much is X?" and X is a model name → models; if X is a plan name → plans; if it resembles neither → first run `pricing plans` to list all four plan tiers and check for a match, then fall back to models.

## `--modality` value mapping (translated by the Agent)

The backend `FoundationModelDomain` field has **five coarse-grained enum values**. arkcli passes them through without translation. When the user says "video / image / text" in natural language, you (the Agent) must translate it before passing the value:

| User may say | Actual backend enum | Note |
|---|---|---|
| Text / LLM / language model | `LLM` | |
| Image / picture / vision | `ComputerVision` | |
| Video | `ComputerVision` | ⚠️ The backend does not distinguish image/video/3D and **returns image and 3D models** |
| 3D / hyper3d | `ComputerVision` | Same as above |
| Audio / voice / TTS / ASR | Do not query | `Audio` is a backend enum, but this arkcli skill does not use it to answer marketplace voice model pricing; only state that it is unsupported |
| Vector / embedding | `Embedding` | |
| Routing / router | `Router` | |

**Important**: When the user asks "How much does video cost?", you should:
1. Run `arkcli pricing models --modality ComputerVision` (do not pass `--modality video` directly).
2. Filter video models from the response (using `DisplayName` or `FoundationModelTag`), or explicitly tell the user that "the backend returns the broad vision category, including image / video / 3D."

## Key `ChargeItem.Type` enum mapping (required for response parsing)

Each base model's `ChargeItems` is **comprehensive** and may mix rates for base inference, fine-tuning training, fine-tuned inference, and generation tasks. The Agent should select only the 1–2 entries relevant to the user's question.

| User question semantics | Type to select |
|---|---|
| Inference price | `InferencePrompt` + `InferenceCompletion` |
| Batch inference price | `BatchInferencePrompt` + `BatchInferenceCompletion` |
| Context cache price | `ContextSessionHit` + `ContextSessionStorage` |
| Fine-tuning price | `Finetune` (full) or `LoraFinetune` (LoRA) |
| Price to invoke a fine-tuned model | `FinetuneInferencePrompt` + `FinetuneInferenceCompletion` |
| Image-to-image price (base model) | `T2ICompletion` / `I2ICompletion` |
| Image-to-image price (fine-tuned model) | `FinetuneT2ICompletion` / `FinetuneI2ICompletion` |
| Video generation | `T2VCompletion` / `I2VCompletion` / `V2VCompletion` (and 1080/4K variants) |
| 3D generation | `To3DCompletion` / `FinetuneTo3DCompletion` |

For the complete Type enum, see "Complete ChargeItem.Type enum mapping" in [`references/arkcli-pricing-models.md`](references/arkcli-pricing-models.md).

## CustomizationType → ChargeItem.Type mapping

The top-level response from `pricing models --model cm-*` includes `resolved_custom_model.customization_type`. Select the rate accordingly. For the complete mapping table, see "Custom model (cm-*) reverse lookup" in [`references/arkcli-pricing-models.md`](references/arkcli-pricing-models.md).

## Quick decisions

- User asks "How much does model X cost?": `arkcli pricing models --model X`; select `InferencePrompt` / `InferenceCompletion` from `ChargeItems`.
- User asks "How much does my cm-xxx cost?": `arkcli pricing models --model cm-xxx`; inspect `resolved_custom_model.customization_type` → select the corresponding `Finetune*` entry using the mapping above; also tell the user its base model from `resolved_custom_model.foundation_model_name`.
- User asks "Prices for all LLMs": `arkcli pricing models --modality LLM`.
- User asks "How much does a TTS / ASR / voice model cost?": do not run pricing; clearly state that pricing queries are currently unsupported.
- User asks "How much is Coding Plan?": `arkcli pricing plans` to list all four tiers, or add `--plan coding-plan-personal-lite` for one tier.
- User asks "What free quota do I have?": `pricing models`; inspect `InferenceFreeUsage` and `ResourcePackItems[Type=FreeInference]`.
- User asks "Does my account have a discount?": compare `Price` and `OriginalPrice`; the difference is the discount (supported by both commands).
- User asks "What is my actual consumption?": this is not a pricing question; route to [arkcli-usage](../arkcli-usage/SKILL.md).
- **No permission for enterprise plans**: `pricing plans` fills the `error` field for the two enterprise rows with `AccessDenied: ...`, while the two personal rows return normally. This is expected behavior, not a bug.

## Common fallbacks

- Authentication error: route to [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).
- The user asks about consumed usage rather than unit prices: route to [`../arkcli-usage/SKILL.md`](../arkcli-usage/SKILL.md).
- The user asks about voice model pricing / TTS pricing / ASR pricing: route to [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md), and only explain that the models can be discovered in the marketplace and that arkcli currently does not support pricing queries.

## References

- [arkcli-usage](../arkcli-usage/SKILL.md) — Actual token / request consumption (including historical ranges)
- [arkcli-deploy](../arkcli-deploy/SKILL.md) — Create an Endpoint (automatically checks activation before deployment)
- [arkcli-shared](../arkcli-shared/SKILL.md) — Authentication and global parameters
