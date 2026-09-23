# models get

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

View model details, aggregating multiple underlying model metadata APIs to return complete information.

## Commands

```bash
# Positional argument form (recommended)
arkcli models get dola-seed-2-1-turbo-260628

# Specify a version
arkcli models get dola-seed-2-1-turbo-260628 260628

# Explicit flag form
arkcli models get --id dola-seed-2-1-turbo-260628 --version 260628
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `<id>` / `--id` | Yes | string | Model identifier, such as `dola-seed-2-1-turbo-260628` |
| `[version]` / `--version` | No | string | Model version override, such as `260628` |

## Return value

Model details in JSON format, aggregated from multiple underlying APIs. Includes model name, version, capabilities, pricing, rate limits, and other information.

`supported_params` is enriched for the exact model version in the detail result. Primary versions may reuse a fresh local ArkModels metadata cache; when that cache is unavailable, for explicit non-primary versions, or with `--no-cache`, the CLI uses exact-version `ListModelMetaDatas` instead. `models get` does not add ArkModels network calls for this enrichment, avoiding its low-QPS limit. `--no-cache` still bypasses both CardView and ArkModels metadata caches.

- Missing field or empty array: no usable parameter catalog is currently available for that version, so `--transform supported_params` may print `null`.
- Present but malformed upstream JSON: the CLI prints `warn: model supported_params enrichment failed: ...` with model/version context to stderr while stdout still returns the remaining model detail.

## Prices and entitlements

Prices now use `pricing.model_name`, `pricing.prices`, and optional `pricing.dimension_attributes`. The old `pricing.charge_items` / `pricing.multi_charge_items` arrays have been removed. Scripts and Agents must migrate; do not filter by the old `type` field.

```bash
# Prices plus existing activation and entitlement fields
arkcli models get dola-seed-2-1-turbo-260628 --format json --transform pricing

# Unified price rows only
arkcli models get dola-seed-2-1-turbo-260628 --format json --transform pricing.prices
```

| Field | Consumption rule |
|------|------------------|
| `service_type` | Distinguishes `infer`, `fast-infer`, `flex-infer`, `batch-infer`, `infer-storage`, `finetuned-infer`, and `finetuned`; never mix training and inference |
| `label` | Billing item identifier, interpreted with its service and conditions; not the old `type` |
| `dimensions` | Full applicability conditions, including windows, periods, and resolution; optional `ranges` describe their scope, and empty values are meaningful |
| `usage_unit` / `unit_code` / `usage_period` | Usage unit, price unit, and optional period; preserve them, do not assume per-thousand tokens or infer currency |
| `price` / `original_price` | Nullable numbers; `null` means unavailable, while `0` is a zero rate for that condition |
| `discount_price_start_time` / `discount_price_end_time` | Optional discount validity times; do not invent missing values |
| `group_index` | Original group position in this response, not a stable ID across requests; do not merge separate groups |
| `dimension_attributes` | Model-level dimension attributes, including those not referenced by price rows |

- Match the service, billing item, and all applicability conditions before displaying a rate. If multiple rows match, show their differences instead of taking the first or cheapest. If a dimension cannot be interpreted, explain the unknown condition and do not estimate a total.
- `pricing.prices: []` means no prices for this query; it does not prove that a model is free, inactive, or nonexistent. Price request failures fail the command without falling back to legacy prices.
- `pricing.state`, `inference_free_usage`, `resource_pack_items`, `sub_services`, and other existing non-price fields retain their paths and legacy entitlement source. Missing entitlements do not mean zero quota or inactive status.
- Prices are queried by the resolved foundation-model name. `--version` is not a price filter. The query uses marketplace dimension defaults and global prices; the profile's physical region is not a pricing dimension.
- This migration applies only to `models get`. `pricing models`, fine-tuning price lookup, and estimation retain their existing contracts; do not apply this schema to their outputs.

## Common errors

| Error | Cause | Handling |
|------|------|---------|
| Model does not exist | The ID is misspelled or the model has been taken offline | Use `arkcli models search` to confirm the model name |
| Authentication failed | Not logged in or credentials expired | Run `arkcli auth login` to re-establish BytePlus identity |

## Notes

- Both `id` and `version` support positional arguments and flags.
- This command aggregates multiple underlying APIs and may be slightly slower than `list`.

## Guards

- If authentication fails, first return to `arkcli auth status`, then run `arkcli auth login`.
- If the model ID is uncertain, first use `arkcli models search` or `arkcli models list --name` to confirm it and avoid repeatedly calling a nonexistent ID.
- Prefer read-only queries. Do not switch profiles or modify local configuration while troubleshooting model details unless the user explicitly confirms.

## References

- [arkcli-models](../SKILL.md) -- All models commands
- [arkcli-shared](../../arkcli-shared/SKILL.md) -- Authentication and global parameters
