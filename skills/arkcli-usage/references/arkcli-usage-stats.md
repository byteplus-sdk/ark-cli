# usage stats

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

> **⚠️ Data freshness:** `usage stats` uses an upstream BFF aggregation pipeline with a **5–30 minute delay** and is intended for daily/aggregate analysis. It is **not suitable** for real-time budget monitoring, rate limits, or alerts. For real-time controls, read per-request `.usage` from inference commands ([arkcli-chat](../../arkcli-chat/SKILL.md) / [arkcli-gen](../../arkcli-gen/SKILL.md)); this data has no aggregation delay and is measured per call.

Query ARK inference usage over a date range with filtering/grouping by endpoint, model, API Key, and other dimensions.

**Voice models are unsupported**: do not query TTS / ASR / dubbing / podcast / voice / real-time voice interaction with `usage stats --model`; state that usage queries are unsupported.

## Commands

```bash
# Query today's usage (--start is required; omitting it returns required flag(s) "start" not set)
arkcli usage stats --start <YYYY-MM-DD>

# Query this month by day
arkcli usage stats --start 2025-09-01 --end 2025-09-30

# Hourly query
arkcli usage stats --start 2025-09-17 --end 2025-09-19 --interval Hour --endpoint ep-20xxxx-xxx

# Query a specified model
arkcli usage stats --start 2025-09-01 --end 2025-09-30 --model dola-seed-2-1-turbo-260628

# Group all usage by model
arkcli usage stats --start 2025-09-01 --end 2025-09-30 --by model

# Group by endpoint
arkcli usage stats --start 2025-09-01 --end 2025-09-30 --by endpoint

# Group by model and endpoint
arkcli usage stats --start 2025-09-01 --end 2025-09-30 --by model,endpoint

# Filter by API Key
arkcli usage stats --start 2025-09-01 --end 2025-09-30 --apikey ak-xxxxxxxx

# Query through a profile that owns the intended project
arkcli usage stats --start 2025-09-01 --end 2025-09-30 \
  --profile platform_ap-southeast-1_my-project

# Show billing-window details
arkcli usage stats --start 2025-09-01 --end 2025-09-30 --show-window-detail

# Entry for "my usage" (frequent AI Agent call)
# Endpoint dimension by default: list all Endpoints I created and query their combined usage
arkcli usage stats --start <YYYY-MM-DD> --mine

# Explicit endpoint dimension (same semantics as --mine)
arkcli usage stats --start <YYYY-MM-DD> --mine --mine-by=endpoint

# APIKey dimension: list all active APIKeys under my account, query each, and merge
arkcli usage stats --start 2026-05-01 --end 2026-05-08 --mine --mine-by=apikey

# Fuzzy match by APIKey suffix (aligned with console AuthToken ValueLike)
arkcli usage stats --start <YYYY-MM-DD> --apikey-suffix d9bcce38dab0
```

## Parameters

| Parameter | Required | Type | Description |
|---|---|---|---|
| `--start` | **Yes** | string | Start date, YYYY-MM-DD |
| `--end` | No | string | End date in YYYY-MM-DD format. The interval from `--start` cannot exceed 31 days; defaults to today |
| `--interval` | No | string | Query granularity: `Day` (default) or `Hour` |
| `--endpoint` | No | string | Filter by endpoint ID |
| `--model` | No | string | Filter by model name; adds `ModelName`, `ModelUnitID` |
| `--apikey` | No | string | Filter by the complete API Key value (`Values` exact matching; retained for compatibility with the earlier behavior) |
| `--apikey-suffix` | No | string | Filter by API Key suffix using `ValueLike` fuzzy matching, consistent with the console; the last 12 characters are recommended |
| `--mine` | No | bool | Restrict results to resources owned by the current authenticated identity; used with `--mine-by` |
| `--mine-by` | No | string | Dimension used by `--mine`: `endpoint` (default) means all Endpoints created by me; `apikey` means all active API Keys under my account |
| `--by` | No | string | Additional grouping dimensions, comma-separated: `model`, `endpoint`, `apikey` |
| `--show-window-detail` | No | bool | Include details split by billing window. Mutually exclusive with `SplitByWindows`, which is currently available only through the raw `arkcli api GetInferenceUsage` call |

### `--mine` behavior

**BytePlus:**
- **Requires SSO sub-user login**: root accounts and AK/SK logins are rejected, with guidance to run `arkcli auth login` again.
- **Mutual exclusion rules**: `--mine --mine-by=endpoint` conflicts with `--endpoint`; `--mine --mine-by=apikey` conflicts with `--apikey` / `--apikey-suffix`.
  - **`--mine-by=endpoint`**: uses the `sys:ark:createdBy` tag to retrieve all Endpoints created by me and query them together (`--page-all` forced).
  - If `data_count=0` (the Endpoints have no invocation records, or the user mainly invokes models directly through API Keys), immediately retry with `--mine --mine-by=apikey`.
  - **Never fall back to a query without `--mine`**; whole-account data is not "my usage".
  - **`--mine-by=apikey`**: lists all active API Keys under my account and issues one serial request per key (`ValueLike` is single-value), then merges the records; `totals` contains the merged total.

### `--by`

`--by` returns and groups by dimensions without filtering:

| Value | Added columns |
|----|------------|
| `model` | `ModelName`, `ModelUnitID` |
| `endpoint` | `ModelEndpoint` |
| `apikey` | `AuthToken` |

`--model` filters one model; `--by model` returns all models grouped by model. Both add `ModelName` and `ModelUnitID`.

## Return value

JSON format:

```json
{
  "fields": [
    { "name": "AccountID", "type": "BIGINT" },
    { "name": "Day", "type": "DATE" },
    { "name": "InputTokens", "type": "BIGINT" },
    { "name": "CacheTokensHit", "type": "BIGINT" },
    { "name": "OutputTokens", "type": "BIGINT" },
    { "name": "TotalTokens", "type": "BIGINT" },
    { "name": "ReqCnt", "type": "BIGINT" },
    { "name": "ImageCount", "type": "BIGINT" }
  ],
  "records": [...],
  "totals": {
    "InputTokens":  "382",
    "OutputTokens": "5317",
    "TotalTokens":  "5699",
    "ReqCnt":       "11",
    "CacheTokensHit": "0",
    "ImageCount":   "4"
  },
  "data_count": 3
}
```

`totals` sums all metric columns across `records`: `InputTokens`, `OutputTokens`, `TotalTokens`, `ReqCnt`, `CacheTokensHit`, and `ImageCount`. Dimension columns (`AccountID`, `Day`, `ModelEndpoint`, `AuthToken`, `ProjectName`, and `BillingStatus`) are **not** summed. To answer "How many tokens did I use today?", read `totals.TotalTokens`; for a dimensional breakdown, use `records`.

Dimension columns appear under these conditions:

| Column | Appears when |
|------|---------|
| `ModelEndpoint` | `--endpoint` or `--by endpoint` |
| `ModelName` | `--model` or `--by model` |
| `ModelUnitID` | `--model` or `--by model` |
| `AuthToken` | `--apikey` or `--by apikey` |

## Common errors

| Error | Cause | Handling |
|---|---|---|
| `not logged in` | Not authenticated | Run `arkcli auth login` |
| `AuthFailure` / 401 | Invalid AK/SK or SSO token | Log in again |
| Range exceeds 31 days | API limit | Split into multiple queries |
| Empty records | No matching data or filters too strict | Retry without filters |

## Notes

- **`--start` is required**: omitting it returns `required flag(s) "start" not set`; the CLI does **not** default to today. If the user provides no date, set `--start` to today and omit `--end` so the query covers the same day.
- **Data is delayed 5–30 minutes** because of the upstream aggregation pipeline; arkcli only passes it through and cannot reduce the delay. For real-time budget, rate-limit, or alerting controls, use per-request `.usage` instead of polling this interface.
- **Voice models are outside this command's supported scope**: do not construct `--model`, `--by model`, or API Key fallback queries for TTS / ASR / voice interaction; state that arkcli does not support voice-model usage queries.
- arkcli automatically filters out `free_for_coding_plan` usage rows and always retains rows that have a `ModelUnitID`.
- With `--interval Hour`, the returned `Hour` field has type STRING.
- Date range cannot exceed 31 days.

## References

- [arkcli-usage](../SKILL.md) -- Usage overview
- [arkcli-chat](../../arkcli-chat/SKILL.md) -- Real-time per-request `.usage`
- [arkcli-gen](../../arkcli-gen/SKILL.md) -- Real-time image/video task `.usage`
- [arkcli-shared](../../arkcli-shared/SKILL.md) -- Authentication and global parameters
