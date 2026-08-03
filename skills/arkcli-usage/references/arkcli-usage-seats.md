# usage seats

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

> **Difference from other commands:**
> - `usage plan` / `balance --type plan`: the **caller's own** subscription balance (a sub-user views their own seat).
> - `usage seats`: **admin view** listing **all seats** in a team.
>
> Sub-users normally receive `AccessDenied`; admin/root SSO is required.

## Commands

```bash
arkcli usage seats --product coding-plan-team \
  [--biz-info lite|pro] \
  [--billing-status Pending|Running|Expired|Reclaimed] \
  [--seat-status Idle|Active] \
  [--user-id <id>] [--user-name <name>] [--seat-id <id>] \
  [--project <project>] [--with-usage]

# List all CodingPlan team seats
arkcli usage seats --product coding-plan-team

# Filter by BizInfo tier
arkcli usage seats --product coding-plan-team --biz-info lite

# Filter by BillingStatus (prepare reclaimed seats for cleanup)
arkcli usage seats --product coding-plan-team --billing-status Reclaimed

# Query the seat bound to a specified sub-user
arkcli usage seats --product coding-plan-team --user-id 12345
arkcli usage seats --product coding-plan-team --user-name alice

# Multiple users (client-side parallel fan-out and merge)
arkcli usage seats --product coding-plan-team --user-name alice --user-name bob --user-name charlie

# Cross-field combination = AND (Alice's seats whose status is Running)
arkcli usage seats --product coding-plan-team --user-name alice --billing-status Running

# Single page (disable auto-pagination)
arkcli usage seats --product coding-plan-team --page-number 1 --page-size 50

# === --with-usage: also retrieve each seat's usage ===
arkcli usage seats --product coding-plan-team --with-usage  # Percent only (CodingPlan backend exposes only percentage)
```

## Parameters

| Parameter | Required | Type | Description |
|---|---|---|---|
| `--product` | Yes | string | Team plan ID: `coding-plan-team`. **Personal plans are unsupported** because they have no seat concept |
| `--biz-info` | No | string | Tier filter: `lite/pro` (CodingPlan) |
| `--billing-status` | No | string slice | Billing status filter (repeatable): `Pending` / `Running` / `Expired` / `Reclaimed` |
| `--seat-status` | No | string | Seat status: `Idle` / `Active`. `Idle` means unbound and available; `Active` means bound |
| `--user-id` | No | string slice | Filter by sub-user ID (repeatable; more than one value triggers client-side parallel fan-out) |
| `--user-name` | No | string slice | Filter by sub-user name (repeatable; more than one value triggers client-side parallel fan-out) |
| `--seat-id` | No | string slice | Specify a SeatID (repeatable; sent as a native wire array in one call) |
| `--project` | No | string | Project filter; defaults to the active profile's project when omitted |
| `--page-size` | No | int | Page size; default 100 |
| `--page-number` | No | int | 1-based page number; **explicitly setting it disables auto-pagination** and retrieves only that page |
| `--sort-by` / `--sort-order` | No | string | Sort field / ascending or descending order |
| `--with-usage` | No | bool | **Team products only**: join each seat's usage. CodingPlan team uses `ListSeatInfoUsages` and returns `coding_usage{periods[session/weekly/monthly with percent/reset_at]}`. `reset_at` is returned in RFC3339 Beijing time (UTC+08:00) |

Unknown seat or billing status values fail validation and never degrade to an unfiltered query.

## Default behavior: auto-pagination

Without `--page-number`, the service layer automatically follows `NextToken` until the last page (maximum 30 pages × 100 = 3,000 seats). Reaching the limit reports an error and directs the user to paginate manually.

## Multi-value filter semantics

Different flags have an **AND** relationship: each flag is a narrowing filter, as in a SQL `WHERE` clause or kubectl `-l`. Multiple values for the same flag have an **OR** relationship.

| Example | Semantics | Wire calls |
|---|---|---|
| `--seat-id sa --seat-id sb` | seat ∈ {sa, sb} | One native `Filter.SeatIDs:["sa","sb"]` call |
| `--user-name a --user-name b` | user = a OR user = b | **Two parallel calls** (client-side fan-out; wire `Filter.UserName` is scalar and does not accept an array) |
| `--seat-id sa --user-name a` | seat ∈ {sa} AND user = a | One call |
| `--seat-id sa --user-name a --user-name b` | seat ∈ {sa} AND user ∈ {a,b} | Two calls, each with same SeatIDs |
| `--user-name alice --billing-status Reclaimed` | user = alice AND billing = Reclaimed | One call |

Equivalent SQL: `WHERE seat_id IN (...) AND user_name IN (...) AND billing_status IN (...)`.

### The union you do not get—intentionally

```bash
# Does not return the union of seat sa plus all of Alice's seats
arkcli usage seats --seat-id sa --user-name alice
# Actual: only sa, and only if it belongs to Alice
```

To request the union of "seat sa **or** Alice's seats", run two commands and merge the results on the client:

```bash
{ arkcli usage seats --product ... --seat-id sa
  arkcli usage seats --product ... --user-name alice; } | jq -s '...'
```

The CLI intentionally avoids implicit OR. Its filter safety contract is that filters can only narrow results; unexpectedly broadening results can cause incorrect statistics or cleanup.

### What fan-out means

You only need this detail when working directly with the wire API. Wire `Filter.UserID / UserName` fields are scalar, and arrays are rejected with `InvalidParameter`. For multiple users, the service layer starts N goroutines, sends one scalar RPC for each user, deduplicates the client-side results by `SeatID`, and merges them. Consequences:

- **Latency** = the slowest of N concurrent requests, not their sum.
- **Backend load** = N times. Dozens of users are acceptable; for hundreds, use `--all` and filter on the client.
- **Consistency** = separate calls have a seconds-level time window, so a race is theoretically possible if a seat is unbound during the query. This is acceptable for admin inventory.
- If any user query fails, the entire request fails and identifies which user failed; no partial results are returned.

## Return value

```json
{
  "items": [
    {
      "seat_id": "seat-001",
      "account_id": "210000",
      "project_name": "default",
      "biz_info": "lite",
      "billing_status": "Running",
      "seat_status": "Active",
      "identity_type": "IAMUser",
      "instance_id": "inst-xxx",
      "order_time": 1700000000000,
      "expired_time": 1717000000000,
      "create_time": 1700000000000,
      "update_time": 1716000000000
    }
  ],
  "total": 30,
  "biz_summaries": [
    { "biz_info": "lite", "total_count": 30, "active_count": 25 }
  ]
}
```

### Fields

| Field | Meaning |
|---|---|
| `seat_id` | Unique seat identifier |
| `biz_info` | Tier |
| `billing_status` | Billing status (`Running` is billed; `Reclaimed` has been reclaimed) |
| `seat_status` | Binding status: `Idle` is unbound and available; `Active` is bound |
| `identity_type` | Bound identity type (e.g. `IAMUser`) |
| `order_time` / `expired_time` | Order / expiration time in epoch milliseconds |
| `total` | Total number of records matching the filters according to the backend |
| `biz_summaries[]` | Summary by tier; for example, 30 `lite` seats purchased and 25 activated |

> **BytePlus seat item does not carry `user_id`, `user_name`, or `bind_count`.** The `identity_type` field indicates the identity class, but the seat listing itself does not expose which named sub-user is bound to each seat, nor the current-month binding count. Consumers that need "which seat belongs to which employee" must correlate `seat_id` with an external identity source (organization roster, IAM API), or filter with `--user-id` / `--user-name` (see next section for filter-side caveats).

## Admin workflows

**Scenario 1**: View the team's overall subscription status

```bash
arkcli usage seats --product coding-plan-team
# Use biz_summaries to see how many seats were purchased and activated in each tier
```

**Scenario 2**: Find reclaimed (expired) seats for processing

```bash
arkcli usage seats --product coding-plan-team --billing-status Reclaimed
```

**Scenario 3**: Find the seats bound to a specific sub-user

```bash
arkcli usage seats --product coding-plan-team --user-name alice
```

The filter narrows the returned list to seats bound to Alice, but the returned rows themselves do **not** carry `user_name` — the caller only sees `seat_id` and cannot use the response alone to confirm "this specific seat belongs to Alice" for a multi-seat result set. If the user needs a seat→employee mapping in the output, correlate with an external identity source.

**Scenario 4**: Check the current state of specified SeatIDs

```bash
arkcli usage seats --product coding-plan-team --seat-id seat-001 --seat-id seat-002
```

## Common errors

| Error | Cause | Handling |
|---|---|---|
| `--product is required and must be a team plan` | Missing or set to a personal plan | Pass `coding-plan-team` |
| `AccessDenied` | Caller is not an admin | Ask an admin to run the command, or have the sub-user use `usage plan` to view their own seat |
| `exceeded auto-paginate limit (30 pages)` | Team has more than 3,000 seats | Paginate manually with `--page-number` |

## Relation to other commands

- Sub-user's own seat → [`usage plan`](arkcli-usage-plan.md) / [`usage balance --type plan`](arkcli-usage-balance.md)
- Account-level model free quota → [`usage balance --type free-quota`](arkcli-usage-balance.md)

## API alignment

- Wire path: `/open/ListSeatInfos` (OpenAPI). `--with-usage` makes an additional call:
  - CodingPlan team → `/open/ListSeatInfoUsages` (SeatIDs are required, collected from `ListSeatInfos`, and passed in batches of no more than 1,000).
- Pagination uses `NextToken` (cursor), not PageNum.
- On BytePlus, `ListSeatInfos` returns seat metadata (`seat_id`, tier, status, lifecycle timestamps) but omits identity-detail fields such as the bound sub-user's ID/name and the current-month binding count. `--user-id` / `--user-name` still filter at the wire level, but the response body does not echo those values back.
