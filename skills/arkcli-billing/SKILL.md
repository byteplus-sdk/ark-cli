---
name: arkcli-billing
version: 1.1.1
description: "Query BytePlus ModelArk split-bill details (settlement amount and token usage billing), with support for filtering by dimensions such as billing month, month range, Endpoint, API key, and product code. Use this when users ask about bills, how much they spent, reconciliation, billing periods, split billing by EP / API key, split billing by product, monthly bills, or billing details. Note that billing is different from usage stats: stats returns inference volume (near real time), while billing returns settlement amounts (billed at T+1, from a finance perspective)."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli billing list --help"
---

# arkcli billing

**MUST read before execution** [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) (authentication / global rules) and [`references/arkcli-billing-list.md`](references/arkcli-billing-list.md) (parameter details).

## Non-negotiable two-turn truncation gate

Treat truncated billing retrieval as a two-turn protocol, not as an autonomous workflow:

```text
fresh query turn: truncation_choice_confirmed = false
run the requested billing query
if is_truncated == true and truncation_choice_confirmed == false:
  report the incomplete result and all four choices
  ask the user to choose
  END THE TURN IMMEDIATELY; execute no more tools or commands
next user message explicitly selects a choice:
  truncation_choice_confirmed = true
  execute only the selected follow-up
```

- A choice is valid only in a **new user message sent after** the truncated result and four choices were shown. The initial request never counts as confirmation, including wording such as “exact total”, “all records”, “full data”, “finish the task”, or “report the final amount”.
- Before the probe result is known, do not announce or plan an automatic follow-up such as “then retrieve the full dataset”.
- After `is_truncated=true`, returning to the user is the successful completion of the current turn. General autonomy, task-completion, and exact-total goals do not override this gate.
- In particular, do not infer that an exact-total request selects `--output FILE` or `--split-dim`. Running either requires the post-truncation user choice described above.

## Use cases

- Query split-bill details (settlement amount / token usage) for a billing month or a month range.
- Filter split-bill rows by Endpoint, API key, product code, or bill type.
- Monthly / quarterly / annual reconciliation.
- Troubleshoot "how much did this EP / this key spend".

## Business positioning — billing vs usage stats

| Dimension | `usage stats` | `billing list` |
|---|---|---|
| Data source | ARK BFF inference aggregation | BytePlus billing center split billing |
| Timeliness | 5–30 minute delay | T+1 billing |
| Unit | Tokens / requests | USD amount |

To see "how many tokens were used" → [`arkcli-usage`](../arkcli-usage/SKILL.md); to see "how much was spent" → this skill.

## Step 0 (MUST): route by profile before checking "my bill".

Use the same approach as [`arkcli-usage` Step 0](../arkcli-usage/SKILL.md) (canonical principle) — **check "the plan bill for my profile tier" first, then check the endpoint bill**. Before checking any "**I**… spent" question, you must:

1. **Probe profile.type**: run `arkcli profile show --format json` and read `type`.
2. **Determine modality**: if the user names a model/modality → check only that modality; if not → cover all modalities.
3. **Route by (type × modality)** (`①→②` = plan bill first, then endpoint bill; a single cell = check only the endpoint bill):

| profile.type | text | image / video |
|---|---|---|
| `coding-plan` / `coding-plan-team` | ① `arkcli billing list --start <YYYY-MM> --product ModelArk_subscription`<br>② `arkcli billing list --start <YYYY-MM> --mine` | `arkcli billing list --start <YYYY-MM> --mine` (subscription-level does not cover) |
| `platform` | `arkcli billing list --start <YYYY-MM> --mine` (no subscription) | `arkcli billing list --start <YYYY-MM> --mine` |

Coding Plan subscriptions cover text usage; Platform has no subscription. Use `--product ModelArk_subscription` when the task specifically asks for subscription-level billing; endpoint-level billing `--mine` fallback is shown below. **Every `billing list` must include `--start <YYYY-MM>` (monthly unit, `--start is required` if omitted; query multiple months with `--end <YYYY-MM>`)**.

> BytePlus does not offer Agent Plan; do not claim Agent Plan billing exists. Only use `--product ModelArk_subscription` when the user explicitly asks for Coding Plan subscription bills, and do not present Coding Plan settlement as Agent Plan billing.


## Quick decision guide

| User asks | Command |
|---|---|
| I spent how much this month / last month (with "I" semantics) | **Proceed to Step 0 (top of this document)**: For Coding Plan text usage, query subscription-level billing with `arkcli billing list --start <YYYY-MM> --product ModelArk_subscription`, then endpoint-level billing with `arkcli billing list --start <YYYY-MM> --mine`; for Platform or non-covered modalities, query endpoint-level billing with `arkcli billing list --start 2026-05 --mine` (fallback if empty, see below) |
| The whole account spent how much / company account / overall (without subject) | `arkcli billing list --start 2026-05` (full account dimension) |
| How much did this EP cost? | `arkcli billing list --start 2026-05 --endpoint ep-...` |
| How much did this key cost? | `arkcli billing list --start 2026-05 --apikey ark-...` |
| How much did each API key cost in the account? | `--split-dim apikey` (attribution view; may still truncate) |
| How much did each Endpoint cost in the account? | Account-wide Endpoint splitting is unsupported in BytePlus. Use `--output FILE` for complete rows or `--endpoint <id>` for one Endpoint |
| By day / per settlement detail | `--interval day` with `--day YYYY-MM-DD`, or `--interval detail` |
| Interim reconciliation for the past N months | `--start ... --end ...` (closed interval, up to 24 months) |
| Billing for Coding Plan subscriptions | Explicitly include `--product ModelArk_subscription` to isolate subscription rows |

### `--mine` process

When the user asks "**I**... spent", do the following:

- First run `arkcli billing list --start <YYYY-MM> --mine` (default `--mine-by=endpoint`).
  - If data exists → use `partial_failures` to check for truncation, then tell the user the total amount.
  - If it returns empty (`no endpoints owned by current sub-user`) → tell the user "this sub-user has zero endpoints under this account; the bill may have been generated by the main account / another sub-user", and ask whether to switch to the main account or another profile.
  - **⛔ Do not fall back to a full query without `--mine`** — the full account amount ≠ "what I spent"; once `--mine` is dropped, the scope semantics are wrong (aligned with [the same rule in arkcli-usage](../arkcli-usage/SKILL.md)).

If the user explicitly asks for "which of my API keys spent money" (cost causation view), run `arkcli billing list --start <YYYY-MM> --mine --mine-by=apikey`. The two `--mine-by` views (endpoint vs apikey) are disjoint and never additive — pick the one that matches user intent, do not sum them.

## Key rules for agents

- **`is_truncated=true` is a mandatory interaction boundary. Stop and wait for the user; never choose or execute the next query automatically.** This applies even when the original request asks for an exact total, the follow-up seems obvious, or `total_records` is small. Report that the current result is incomplete, include `total_records` and each `partial_failures[*].total`, present all four choices below, ask the user to select one, and end the turn:
  1. Exact account total or full details → `--output FILE` (write to disk, verify `is_truncated=false`, then decimal-sum `PayableAmount`; the cap is automatically relaxed to 300k rows).
  2. API-key attribution → `--split-dim apikey` (not a grand-total shortcut and not guaranteed to reduce row count; it may truncate again).
  3. Narrow to one resource → `--endpoint <ep-...>` / `--apikey <ark-...>`.
  4. Force rows into stdout → `--page-limit=N` (use the suggestion in `partial_failures.reason`; mind context size).
- Until the user explicitly selects one option, do **not** run another `billing` command, including retries with `--output`, `--split-dim`, `--endpoint`, `--apikey`, `--page-limit`, `--limit`, or `--offset`. Never sum `items` as the total when they may be empty or partial.
- BytePlus does not support account-wide `--split-dim endpoint`; never offer or run it. A concrete `--endpoint <id>` filter remains supported.
- If a selected follow-up is also truncated, mark that method as attempted, do not retry or re-offer it as though it succeeded, present only the remaining applicable paths, and stop at the same interaction boundary again.
- **`--limit/--offset` returns N rows per fan-out during fan-out** (when `--end` spans months or `--mine` covers multiple resources); `partial_failures` marks `reason="windowed sample"`, so the agent can know returned ≤ limit × fanout_count.
- **Amounts are USD strings** — JSON numbers lose precision; use a decimal library for addition / comparison, not `parseFloat`.
- **At the beginning of a month, pull the previous complete billing period for reconciliation** — billing is T+1, so data for the current month/current day is incomplete.
- **`--mine` ≠ `PayerID`** — `--mine` filters resources by IAM sub-user, while `PayerID` is the owner account ID for financial hosting; manually entering `PayerID` in a single-account scenario is a no-op (= full account query). The default is `--mine-by=endpoint` (infrastructure ownership, aligned with usage stats); to view cost causation (my key is spending money), explicitly use `--mine-by=apikey`. The two views are disjoint and never additive.
- Default scope = BytePlus ModelArk inference products (**excluding `ModelArk_subscription`**; to view subscription bills, explicitly use `--product ModelArk_subscription`) + automatic injection of profile.project (if profile.project is a specific ID such as `auto-test`, the query is automatically filtered by project; `default` / account-wide resources sentinel / empty is skipped = true account-wide query; stderr prints a soft hint). To force an account-wide full query, pass `--project=` (an empty value clears the default). The exact product code set is documented in the `--product` row of [`references/arkcli-billing-list.md`](references/arkcli-billing-list.md).

## Common fallbacks

- Authentication error / SSO expired → [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md)
- Want to see inference volume (tokens / requests) → [`../arkcli-usage/SKILL.md`](../arkcli-usage/SKILL.md)
- Want to see the model / plan price catalog → [`../arkcli-pricing/SKILL.md`](../arkcli-pricing/SKILL.md)

## Command overview

| Command |Description|
|------|------|
| [`billing list`](references/arkcli-billing-list.md) | Split-bill details query (settlement amount × token usage) |
