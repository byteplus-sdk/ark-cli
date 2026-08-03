# usage plan

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

> **Difference from `stats` / `billing`:**
> - `usage stats`: token-billed **inference usage** (time series, 5–30 minute delay).
> - `billing list`: monthly **settlement amount** (BytePlus billing-center breakdown, T+1).
> - `usage plan`: **subscription plan quota snapshot** for CodingPlan (real-time "My plans" backend data, matching BytePlus ModelArk console 1:1).
>
> "How many tokens/requests did I use?" → stats; "How much money?" → billing; "How much plan quota remains / percentage used / reset date?" → **plan**.

Query subscription quota snapshots. A plan is not queried as a time series; it returns **used / total / reset time for the current effective subscription by window (5h / weekly / monthly / session)**.

## Commands

```bash
# Default: concurrently probe both products (personal × 1 + team × 1);
# issue usage requests only for detected buckets, returning all active subscription balances
arkcli usage plan

# Explicit product (skip discovery and query one bucket)
arkcli usage plan --product coding-plan
arkcli usage plan --product coding-plan-team

# Force both buckets (personal + team) regardless of subscription (diagnostic / Excel view)
arkcli usage plan --all

# Query a specified team seat (bypass automatic caller-seat resolution through GetSeatInfo)
arkcli usage plan --product coding-plan-team --seat seat-001
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `--product` | No | string | Explicit product ID: `coding-plan` / `coding-plan-team`; omitted means automatic subscription discovery |
| `--all` | No | bool | Force both products, including unsubscribed buckets. **Mutually exclusive with `--product`** |
| `--seat` | No | string | Team product only. Omitted: service calls `GetSeatInfo` to find caller seat; provided: skip RPC and use this SeatID (admin view) |

## Default behavior (no `--product` / `--all`)

Without a product, behavior is unified (`profile.Type` no longer derives one bucket; since V2, all subscriptions are probed because one identity may hold multiple products):

1. Probe the two buckets' subscription status concurrently:
   - **Personal × 1** via `ListSubscribeTrade` (`ResourceNames=[""]`).
   - **Team × 1** via `GetSeatInfo(Scene="")`. The enterprise edition is not exposed through `ListSubscribeTrade` and returns null in testing, so caller-seat binding status is used instead.
2. Send usage RPC for detected buckets:
   - `coding-plan` personal → `GetCodingPlanUsage`.
   - `coding-plan-team` → use resolved SeatID → `GetSeatInfoUsage(SeatID=id, Scene="")`.
3. Failure in one bucket does not block others (per-bucket error isolation).

> **One extra RPC for zero-flag UX**: concurrent `ListSubscribeTrade × 1 + GetSeatInfo × 1` takes ~50 ms, much less than querying products serially. Pass `--product` to skip discovery.
>
> **Silent team-bucket rule**: if caller has no seat, the team bucket does **not appear** in default discovery, preventing misleading "no seat bound" for admins without enterprise purchase. Use `--all` to show both buckets including the error row.

## Return value

JSON shape (top-level `viewer` + `items`):

```json
{
  "viewer": {
    "auth_method": "sso",
    "is_root": false,
    "user_id": "12345",
    "user_name": "alice",
    "account_id": "210000",
    "profile": "my-profile",
    "tenant": "byteplus",
    "region": "ap-southeast-1",
    "project_name": "default"
  },
  "items": [
    {
      "product": "coding-plan",
      "edition": "personal",
      "tier": "lite",
      "subscribed": true,
      "periods": [...]
    },
    {
      "product": "coding-plan-team",
      "edition": "team",
      "tier": "lite",
      "seat_id": "seat-001",
      "subscribed": true,
      "periods": [...]
    }
  ]
}
```

### `viewer` fields (identity summary)

```
viewer.auth_method  = sso | aksk | apikey | none
viewer.is_root      = SSO root account (JWT trn contains :root)
viewer.user_id      = SSO sub-user ID (normally empty for AK/SK / APIKey)
viewer.user_name    = SSO sub-user name
viewer.account_id   = JWT sub claim
viewer.profile      = current active profile name
viewer.tenant       = byteplus
viewer.region       = active region
viewer.project_name = active project
```

**Purpose**: Sub-user and root views are easily confused. `viewer` tells the user/Agent "whose data this is":
- Sub-user SSO → `is_root=false`; user_id/name identify the sub-user, and the view shows that user's bound seat.
- Root SSO → `is_root=true`; root/account subscription view.
- AK/SK → `auth_method=aksk`, user_id normally empty; backend resolves IAM identity from AK/SK.
- APIKey → `auth_method=apikey`; scope follows the identity bound to the API Key.

### Field descriptions

| Field | Description |
|---|---|
| `product` | `coding-plan` / `coding-plan-team`, matching `profile.Type` |
| `edition` | `personal` / `team` |
| `tier` | Actual subscription tier (`lite/pro`) returned by backend; CodingPlan often absent |
| `subscribed` | Whether current identity has an active subscription; for CodingPlan, `QuotaUsage` non-empty |
| `periods[].label` | CodingPlan: `session` / `weekly` / `monthly`, sorted canonically |
| `periods[].used` / `total` | The CodingPlan backend returns only `Percent`; `used` / `total` are omitted through JSON `omitempty` |
| `periods[].percent` | Percentage used (0–100). CodingPlan passes the backend `Percent` through directly |
| `periods[].reset_at` | Next reset time in **RFC3339 Beijing time (UTC+08:00)**. Internally, the service normalizes backend seconds to epoch milliseconds before the output layer formats them; a no-data sentinel is omitted |
| `updated_at` | CodingPlan only; backend `UpdateTimestamp` epoch ms |
| `error` | Per-bucket failure reason; one bucket does not block others. A team-product `NotImpl` error also uses this field |

## Error handling

- **AccessDenied/network**: written to corresponding `items[].error`; other buckets unaffected.
- **Unsubscribed**: empty CodingPlan `QuotaUsage` → `subscribed:false`.
- **Team subscribed but caller has no seat**: `subscribed:true` + `error:"no seat bound to caller for ..."`; pass `--seat <id>` to bypass.
- **Product typo**: outside the `coding-plan` / `*-team` set → `ErrValidation` immediately, without sending a request.
- **`--all` + `--product`**: mutually exclusive ErrValidation.
- **All auto-discovery fails**: returns `auto-discover subscriptions: <err>` (for example, backend rejection such as `MissingParameter.ResourceNames`); use explicit `--product`.

## AI Agent decision path

- "How much plan used/remains?" → `arkcli usage plan`.
- "How many plans do I have?" → `arkcli usage plan --all`; buckets with `subscribed=true` are held.
- "Admin view of another team seat" → `arkcli usage plan --product coding-plan-team --seat <id>`.
- "Why are plan items empty?" → probably no products under current identity; route to [arkcli-pricing](../../arkcli-pricing/SKILL.md) for catalog pricing.

## API alignment

- CodingPlan personal: `/open/GetCodingPlanUsage`.
- CodingPlan team: `/open/GetSeatInfoUsage`.
- Auto-discovery uses a `ListSubscribeTrade` payload for personal × 1. The enterprise edition is not exposed through `ListSubscribeTrade`, so `GetSeatInfo(Scene)` probes the caller's seat-binding status instead.
- Team SeatID resolution: `/open/GetSeatInfo` without a SeatID resolves the bound seat from caller identity + Scene. Scene **must be an empty string** (CodingPlan enterprise; this is the backend default). Early use of `coding_plan` returned an empty SeatID without an error.
