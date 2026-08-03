# billing list

> **Prerequisite**: Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first (authentication / global parameters).

> **`billing list` vs `usage stats`**: stats shows "how many tokens were used" (near real time, aggregated by ARK BFF); billing shows "how much money was spent" (T+1 billing, BytePlus Billing Center).

Query split bill details for a specified billing period (YYYY-MM) or month range.**Default full pagination**. If the service internal cap (~3,000 rows / 1 MB JSON) is hit, **soft truncation** is used (no hard fail): the data already fetched is returned normally + `is_truncated=true` + `partial_failures` marks which fan-outs were truncated + a strong warning is printed to stderr.`--limit/--offset` switches to explicit pagination mode (N rows per fan-out when spanning fan-outs).

**Core discipline**: check `is_truncated` and `partial_failures` before using any returned row. When `is_truncated=true`, the result is incomplete and the agent must not sum it or choose a follow-up itself. Report the scale, present all four choices documented below, ask the user to select one, and stop. Run a follow-up command only after the user explicitly chooses.

**Two-turn requirement**: the user choice must arrive in a new message after the agent has displayed the truncated result and four choices. Intent expressed in the original query—including “exact total”, “all records”, or an explicit desired final answer—is not a choice and cannot authorize `--output`, aggregation, pagination, or narrowing. Do not pre-plan a second command before the first result. When the first result is truncated, end the turn immediately after asking the question.

> **Using `--start` alone = single-month query** (not an open interval through today).`billing list` uses month-level granularity, and the wire has a single `BillPeriod` parameter — to query a multi-month range, you must use `--end` (for example: `--start 2026-03 --end 2026-05`, simulated by client fan-out). This is different from the day-level `--start --end` in `usage stats`.

## Commands

```bash
# Single-month summary (most common) — default: BytePlus ARK product allowlist (excluding subscriptions) + profile.project
# Automatically injected (if profile.project is a specific ID, such as "auto-test"); profile.project="default" /
# Account-wide resource sentinel / empty value skips injection = real account-wide query. A soft hint is printed to stderr.
arkcli billing list --start 2026-05

# Multi-month range (closed interval, up to 24 months)
arkcli billing list --start 2026-03 --end 2026-05

# By day / by each settlement detail
arkcli billing list --start 2026-05 --interval day --day 2026-05-15
arkcli billing list --start 2026-05 --interval detail

# Single-value scope (choose one of three, mutually exclusive)
arkcli billing list --start 2026-05 --endpoint ep-...
arkcli billing list --start 2026-05 --apikey ark-...        # Automatically reverse-looks up the SID through the server
arkcli billing list --start 2026-05 --apikey-sid apikey-... # advanced, skips reverse lookup

# Account-dimension slicing (API-key attribution only)
arkcli billing list --start 2026-05 --split-dim apikey      # Which API keys incurred cost; may still truncate

# BytePlus does not support account-wide --split-dim endpoint.
# To inspect one Endpoint, use the explicit resource filter instead:
arkcli billing list --start 2026-05 --endpoint ep-...

# My own costs (from the SSO subuser perspective)
arkcli billing list --start 2026-05 --mine                  # Defaults to --mine-by=endpoint (infra ownership)
arkcli billing list --start 2026-05 --mine --mine-by=apikey # Cost-causation view (which of my API keys spent money)

# Other filters
arkcli billing list --start 2026-05 --product ModelArk_subscription  # Isolate subscription bills
arkcli billing list --start 2026-05 --bill-category consume-use --billing-mode 2
arkcli billing list --start 2026-05 --ignore-zero
```

## Parameter

| Parameter | Required |Description|
|------|---|------|
| `--start` | **Yes** | A single billing period `YYYY-MM` (within the past 24 months) |
| `--end` | No| Range endpoint `YYYY-MM` (closed interval, omitted = single month, up to 24 months) |
| `--day` | No| Billing date `YYYY-MM-DD`, **only `--interval=day\|detail` takes effect**, incompatible with `--end` |
| `--interval` | No| `month` (default) / `day` / `detail` |
| `--endpoint` / `--apikey` / `--apikey-sid` | No| Single-value scope, mutually exclusive among the three options (single SplitItemID value).`--apikey ark-*` is automatically reverse-looked up into a SID through the server; `--apikey-sid` skips reverse lookup |
| `--split-dim` | No| Account-dimension slicing: `apikey` only in BytePlus. This is an attribution view, not a guaranteed row-reduction or grand-total shortcut. Account-wide `endpoint` is rejected; use `--endpoint <id>` for one Endpoint |
| `--product` | No| BytePlus product code (repeatable). The default lock covers the API-key-splittable BytePlus ModelArk products in the "Product codes" section below; subscriptions = `ModelArk_subscription` (excluded from the default; pass explicitly to include) |
| `--project` | No| Project ID (repeatable).**If not passed, the profile ProjectName is automatically used as the default value**; `--project=` (empty) = account-wide query |
| `--instance-no` | No| Billing instance ID |
| `--bill-category` | No| `consume-use` / `consume-new` / `consume-renew` / `consume-formalize` / `consume-modify` / `consume-trial` / `refund-terminate` / `refund-modify` / `transfer-manual` / `transfer-system` |
| `--billing-mode` | No| `1` subscription / `2` pay-as-you-go / `3` contract / `4` fulfillment |
| `--ignore-zero` | No| Skip rows with discounted price = 0 |
| `--mine` | No| View only bills for resources under the current SSO subuser. The dimension is determined by `--mine-by`.**Mutually exclusive with `--endpoint` / `--apikey` / `--apikey-sid` / `--split-dim`**. Subscriptions are not included (no IAM ownership) |
| `--mine-by` | No| `endpoint` (default, **infra ownership** — bills on EPs I deployed, including calls made by others) / `apikey` (**cost causation** — bills generated by keys I created). The two views cannot be added together. The former aggregates by infra, and the latter aggregates by payer |
| `--limit` / `--offset` | No| Pagination mode (Limit 1-300, hard wire upper limit), returns a single page directly. During fan-out, this means **N rows per fan-out** (multi-month `--end` / multiple resources with `--mine`), and `partial_failures[]` marks `windowed sample` |
| `--output` | No| Write the complete JSON (items + summary + partial_failures) to FILE, leaving only metadata on stdout to avoid blowing up the context. Automatically relaxes the internal cap to 300k rows |

## Scope view capabilities

Different scopes cover different bill subsets. **`ModelArk_subscription` (Coding Plan) can only be viewed at the account dimension. It is not visible in any IAM/EP/apikey dimension** (account-level ownership, no IAM owner).

| View | ModelArk inference (default lock) | Subscriptions (`ModelArk_subscription`) |
|---|---|---|
| Bare run (default) | ✅ | ❌ (default Product lock filter) |
| `--product ModelArk_subscription` | ❌ | ✅ |
| `--product ModelArk --product ModelArk_subscription` | ✅ (only the specified subset) | ✅ |
| `--endpoint` / `--apikey` / `--apikey-sid` / `--split-dim apikey` / `--mine` | Respective scopes | ❌ |

**Cross-product coverage of `--mine-by=endpoint`**: SplitItemID `ep-...` is a cross-product unique primary key. Bills for endpoints created by `+deploy` are placed under different Product parameters depending on modality (LLM / image / video / managed agents / third-party, see the "Product codes" table); the service layer does not lock Product for scope-narrowed queries, allowing the wire to naturally hit across products by SplitItemID. Therefore, `--mine --mine-by=endpoint` also includes bills for image / video EPs. The same consumption appears once in each of the two mine views, so **they cannot be added together**.

**`--mine` ≠ `PayerID`**:
- `--mine` filters by **IAM subuser** UserID (the client first lists my resource IDs, and the service layer performs per-resource fan-out).
- The wire `PayerID` / `OwnerID` is the **account owner ID** (10-digit integer), used in financial hosting scenarios.
- In a single-account scenario, manually writing `--params '{"PayerID":[<my account ID>]}'` is a no-op (= account-wide query); use `--mine` to view your own data.

## Product codes (BytePlus default lock)

The default lock covers these BytePlus ModelArk inference products. Bare `billing list` runs are filtered to this set; to include subscriptions or other products, override with `--product`.

| Product code | Description |
|---|---|
| `ModelArk` | LLM inference |
| `ModelArk_open_source_llm` | Open-source LLM inference |
| `ModelArk_video_generation` | Video generation |
| `ModelArk_managed_agents` | Managed agents |
| `Smart_Drawing_T2I` | Image generation (text-to-image) |
| `hitem_third_party` | Third-party integration |
| `rodin_third_party` | Third-party integration |

Subscription (`ModelArk_subscription`, Coding Plan) is a separate product excluded from this default; pass `--product ModelArk_subscription` explicitly to view it (see "Scope view capabilities" above).

## Return values

```json
{
  "items": [
    {
      "BillPeriod":         "2026-05",
      "BillingDate":        "2026-05-15",
      "Product":            "ModelArk",
      "ProductZh":          "BytePlus ModelArk LLM inference",
      "SplitItemID":        "ep-...",
      "SplitItemName":      "seed-pro-4k-prod",
      "Currency":           "USD",
      "OriginalBillAmount": "123.4500",
      "PreferentialAmount": "10.0000",
      "PayableAmount":      "113.4500",
      "Count":              "1234567",
      "Unit":               "thousand tokens"
    }
  ],
  "total_records": 87881,
  "returned": 3000,
  "is_truncated": true,
  "partial_failures": [
    {
      "period": "2026-05",
      "total": 87881,
      "returned": 3000,
      "reason": "exceeds 10 pages cap (3000 rows scanned, total 87881 rows). Options: --output FILE for complete data; --split-dim apikey for API-key attribution (may still truncate); --endpoint/--apikey to narrow scope; --page-limit=N to raise the cap"
    }
  ]
}
```

| Parameter | Meaning |
|------|------|
| `BillPeriod` / `BillingDate` | Billing period / billing date (BillingDate may be empty when interval=month) |
| `Product` / `ProductZh` | Product code / Chinese name |
| `SplitItemID` / `SplitItemName` | Split item ID / name (endpoint or apikey, depending on the scope) |
| `Currency` | Usually `USD` |
| `OriginalBillAmount` | Amount before discounts |
| `PreferentialAmount` | Discount / resource pack deduction amount |
| `PayableAmount` | **Payable amount** (= Original - Preferential, primary reconciliation parameter) |
| `Count` / `Unit` | Metered value / unit |
| `total_records` / `returned` / `is_truncated` | summary header: actual total count on the server / rows returned this time / whether it is truncated |
| `partial_failures` | Soft truncation / sample marker array (hit the cap or windowed); each item contains `period` / `resource` / `total` / `returned` / `reason`. **Must check when `is_truncated=true`** |

> Amount fields are strings. Use a decimal library for addition/comparison, not `parseFloat` (JSON number precision is lossy).

## Soft truncation (cap / windowed) decisions

When hitting the cap or explicitly using `--limit/--offset`, the service **does not hard fail**. It returns fetched data + a summary header + a strong warning in stderr. Nevertheless, `is_truncated=true` is a mandatory interaction boundary: report the incomplete result, present all four choices, and wait for the user. Do not pick or execute a recommendation automatically, even for a small result set or an exact-total request.

Use this state test before every follow-up:

```text
may_continue = previous assistant turn reported this truncation and asked for a choice
               AND current user message explicitly selected that choice
```

If `may_continue` is false, no second billing command or local post-processing command is allowed. The original request and the agent's own plan cannot make it true.

Present these four choices in this order:

1. Exact account total or full details with `--output FILE`.
2. API-key attribution with `--split-dim apikey`; explain that it may still truncate and is not a grand-total shortcut.
3. Narrow to one resource with `--endpoint` or `--apikey`.
4. Force rows into stdout with `--page-limit=N`.

Never offer account-wide `--split-dim endpoint` in BytePlus. If a selected method also returns `is_truncated=true`, identify it as already attempted, remove it from the actionable choices, present the remaining paths, and wait again. Do not automatically switch from API-key attribution to another method.

| `partial_failures[].reason` | Meaning | Recommended handling |
|---|---|---|
| `exceeds N pages cap (...)` | The default full-fetch mode hits the service fallback cap | Offer the four choices above and wait. Do not automatically rerun with any of them |
| `windowed sample (limit=L, offset=M); returned X of Y rows` | The request uses `--limit/--offset`, so the rows are only a sample | Offer the same four choices and wait. Client-side pagination is allowed only after the user explicitly selects full details or stdout pagination |

For cross fan-out (`--end` across months / `--mine` multiple resources), `--limit/--offset` is **per fan-out**, N rows each (returned ≤ limit × fanout_count); each fan-out has its own item in `partial_failures`, marked with the `period` / `resource` parameters.

## --output FILE mode

Write large data to disk to avoid flooding the agent context through stdout. Consistent with `train finetune logs/metrics/trajectory` `--output`:

```bash
arkcli billing list --start 2026-05 --output bills.json
#   ↳ stdout: {total_records: 87881, returned: 87881, is_truncated: false, output_file: "bills.json"}
#   ↳ bills.json: complete JSON (including 87881 item rows + summary + partial_failures)
```

- The service automatically raises the internal cap to 1000 pages (300k rows / ~100 MB memory), covering full retrieval for most large accounts.
- stdout contains only metadata (items=null + output_file path) — safe for the agent context.
- An explicitly specified `--page-limit=N` is still respected (user-specified value takes priority).
- If it still times out, `partial_failures` is marked; further raise `--page-limit`.

## Client-side pagination loop pattern

Use these pagination patterns only after the user explicitly chooses to retrieve full data. Never start them automatically after observing `is_truncated=true`.

**Fetch the full result for a single month with pagination**:
```bash
total=$(arkcli billing list --start 2026-05 --limit 1 2>/dev/null | jq '.total_records')
offset=0; all="[]"
while [ "$offset" -lt "$total" ]; do
  page=$(arkcli billing list --start 2026-05 --limit 300 --offset "$offset" 2>/dev/null | jq '.items')
  all=$(jq -n --argjson a "$all" --argjson b "$page" '$a + $b')
  offset=$((offset + 300))
done
echo "$all" | jq 'length'   # Should equal total_records
```

**Fetch full results for each month with nested cross-month pagination** (outer fan-out + inner +offset):
```bash
for m in 2026-03 2026-04 2026-05; do
  total=$(arkcli billing list --start "$m" --limit 1 2>/dev/null | jq '.total_records')
  offset=0; items="[]"
  while [ "$offset" -lt "$total" ]; do
    page=$(arkcli billing list --start "$m" --limit 300 --offset "$offset" 2>/dev/null | jq '.items')
    items=$(jq -n --argjson a "$items" --argjson b "$page" '$a + $b')
    offset=$((offset + 300))
  done
  jq -n --arg m "$m" --argjson t "$total" --argjson it "$items" '{month:$m, total:$t, items:$it}'
done | jq -s '.'
```

Client-side pagination is **not** limited by the `--page-limit` cap (the cap is triggered only in the default full-fetch mode).

## Common errors

| Error | Handling |
|------|------|
| `--start is required` / `must be YYYY-MM` | Add or correct to `--start 2026-05` (within the past 24 months) |
| `--end ... must be >= --start` / `range exceeds 24 months` | Adjust the range |
| `--day ... incompatible with --start/--end range` / `requires --interval=day or detail` | Use a single month and a day in that same month, or remove `--day` |
| `--endpoint, --apikey, --apikey-sid are mutually exclusive` | Choose one of the three (single SplitItemID value) |
| `account-wide --split-dim endpoint is not supported` | Use `--endpoint <id>` for one Endpoint or `--output FILE` for complete account data |
| `--split-dim ... incompatible with --endpoint/--apikey` | Use `--split-dim apikey` only for account-wide API-key attribution; otherwise omit it |
| `--apikey value too short` (<16) | Use the full ark-* value, or if you only have the SID, use `--apikey-sid`. |
| `couldn't resolve API key` | Switch to the correct profile, or pass `--apikey-sid` directly. |
| `not logged in` / SSO expired | `arkcli auth login` |

> **Hitting the cap is not listed in the error code table** — soft truncation uses the normal return path (ok=true + is_truncated=true + partial_failures + stderr WARN). See the “Soft truncation decision” section above.

## Notes

- **T+1 billing**: Data for the current month/day may be incomplete. For monthly reconciliation, pull the **previous complete billing period** at the beginning of the month.
- **Large data volume**: ARK bills for a single billing period often have tens of thousands of rows. The default cap is about 3,000 entries (about 1 MB of JSON). Use `--output FILE` for an exact grand total or full details; use `--split-dim apikey` only for attribution, `--limit` for samples, and `--page-limit` only when the user explicitly accepts large stdout output.
- **Amounts are USD strings**: Use a decimal library, not `parseFloat`.
- **Defaults to the API-key-splittable BytePlus ModelArk product set**: Prevents noisy results from unrelated or unsupported split-billing products. **Does not include `ModelArk_subscription`** — to view subscriptions, explicitly specify `--product ModelArk_subscription`. Exact code set: see the "Product codes" section above.

## References

- [arkcli-billing](../SKILL.md) — billing skill overview.
- [usage stats](../../arkcli-usage/references/arkcli-usage-stats.md) — inference usage (tokens / requests).
- [arkcli-shared](../../arkcli-shared/SKILL.md) — Authentication and global parameters
