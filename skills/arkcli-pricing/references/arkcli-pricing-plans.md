# pricing plans

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

List CodingPlan subscription prices (personal + enterprise, four tiers in total). `Price` is the **final subscription unit price (including discounts)** calculated by the backend for the current account; `OriginalPrice` is the published list price.

`pricing plans` and `pricing models` are **two completely independent commands**:
- `pricing models` — Token-based billing (pay-as-you-go model invocation)
- `pricing plans` — Monthly subscription (complete plan)

## Commands

```bash
# List all four plan tiers (personal 2 + enterprise 2 = 4)
arkcli pricing plans

# Query a single tier
arkcli pricing plans --plan coding-plan-personal-pro

# Filter by product
arkcli pricing plans --product coding-plan       # CodingPlan only, four tiers in total

# Filter by edition
arkcli pricing plans --edition personal          # Personal edition only, two tiers in total
arkcli pricing plans --edition enterprise        # Enterprise edition only, two tiers in total

# Combine filters
arkcli pricing plans --product coding-plan --edition personal   # Two CodingPlan personal tiers
```

## Complete Plan ID enumeration

| Plan ID | Product | Edition | Tier |
|---|---|---|---|
| `coding-plan-personal-lite` | CodingPlan | personal | lite |
| `coding-plan-personal-pro` | CodingPlan | personal | pro |
| `coding-plan-enterprise-lite` | CodingPlan | enterprise | lite |
| `coding-plan-enterprise-pro` | CodingPlan | enterprise | pro |

`--plan` accepts the four IDs above (three-part kebab-case). If misspelled, the command returns a structured error and lists all valid IDs.

## Output fields

```json
{
  "items": [
    {
      "plan_id": "coding-plan-personal-lite",
      "product": "CodingPlan",
      "edition": "personal",
      "tier": "lite",
      "price": 10,
      "original_price": 10,
      "period": "monthly",
      "currency": "USD"
    },
    {
      "plan_id": "coding-plan-enterprise-lite",
      "product": "CodingPlan",
      "edition": "enterprise",
      "tier": "lite",
      "price": 0,
      "original_price": 0,
      "period": "monthly",
      "currency": "USD",
      "error": "AccessDenied: caller lacks permission"
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `plan_id` | Three-part ID that can be used as the `--plan` argument |
| `product` | `CodingPlan` |
| `edition` | `personal` or `enterprise` |
| `tier` | Tier identifier; personal/enterprise share the same set (lite/pro) |
| `price` | Final subscription unit price for the current account (after backend settlement) |
| `original_price` | Published list price |
| `period` | Subscription period, currently fixed to `monthly` |
| `currency` | Currency, currently fixed to `USD` |
| `error` | Reason why this row's query failed. Populated for **this row only**; other rows are unaffected |

## Error isolation mechanism

The backend separates "personal batch" and "enterprise batch" into two Actions, so success / failure for personal and enterprise editions is independent:

- Your account has no enterprise-edition permission → the backend returns `AccessDenied`.
- Result: the two enterprise rows have an error message in `error` and `price = 0`; the two personal rows return prices normally.
- The command itself **does not fail**. `arkcli pricing plans` still exits with code 0, and the JSON output contains all four rows.

This matches the BytePlus ModelArk console UI: it also has two tabs (Coding Plan / Coding enterprise edition), and each tab queries pricing and fails independently. A failure in one does not affect the other.

**When the Agent processes the response**:
1. First identify the tier the user asked about → read that row's `price` directly.
2. If that row's `error` is non-empty → tell the user, "The account does not have permission for edition X; contact an administrator."
3. Do not invalidate prices in other rows merely because one row has an `error`.

## Parameter details

### `--plan <plan-id>`

Exactly matches one of the four canonical IDs. Match → query only that tier (only one API call is needed). Misspelling → error message + list of all valid IDs.

### `--product coding-plan`

Also accepts PascalCase (`CodingPlan`). Other values → return an empty items array (no error, because the known-value set is closed, but the Agent workflow is not blocked).

### `--edition personal | enterprise`

Exact match.

### Combination rules

- No flags → all four tiers.
- `--plan X` has the highest priority (when plan is supplied, other flags are ignored because the plan uniquely identifies the product + edition + tier).
- `--product` + `--edition` can be combined (logical AND).

## Comparison with other commands

- User asks "How much is DeepSeek?", "How much is image-to-image?", or "How much does fine-tuning cost?" → [`pricing models`](arkcli-pricing-models.md).
- User asks "How much is Coding Plan small?" or "Personal / enterprise pricing" → this command.
- User asks "How much have I **already** used / consumed?" → [`../arkcli-usage/SKILL.md`](../../arkcli-usage/SKILL.md).

## Backend Actions involved (for raw API troubleshooting)

| Action | Used for |
|---|---|
| `EstimateSubscribePrice` | One CodingPlan personal tier (called once per tier) |

If `pricing plans` completely fails to respond, use the raw API for self-checking:

```bash
arkcli api trade.estimate_subscribe_price --params '{
  "Items": [
    {"BizInfo":"lite","ResourceType":"CodingPlan","ResourceName":"RealCodingPlanPersonal","Period":"monthly","Times":1}
  ]
}'
```
