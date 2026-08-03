# usage balance

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

> **Difference from `stats` / `plan`:**
> - `stats`: **consumption view** (tokens/calls already used).
> - `plan`: subscription quota **snapshot** (used / total / reset).
> - `balance`: **balance view**, unified entry for "how much X remains".
>
> Use `balance` when the user asks "how much X remains", routing by `--type`.
>
> **`balance --type plan` and `usage plan` expose the same data** (both call `QueryPlanUsage`, probe two SKUs, and share flags). Projection differs:
> - `balance --type plan`: concise projection (removes `subscribed` / `updated_at` and adds a `"not subscribed"` hint).
> - `usage plan`: complete projection (includes `subscribed: true/false`, CodingPlan `updated_at`, and other metadata).
>
> Use `balance --type plan` for a balance-oriented view together with free quota or media-asset capacity. Use `usage plan` for a complete plan quota snapshot together with stats.

`arkcli usage balance` is a dispatcher; `--type` selects the data source and is **required**.

## Commands

```bash
# Free model inference quota/resource-pack balance (ListModelChargeItems)
arkcli usage balance --type free-quota
arkcli usage balance --type free-quota --model dola-seed-2-1-turbo
arkcli usage balance --type free-quota --modality LLM --page-size 20

# Media asset library capacity (/open/GetAssetQuota)
arkcli usage balance --type media-asset
arkcli usage balance --type media-asset --project my-project

# Subscription plan balance (QueryPlanUsage, concise output)
arkcli usage balance --type plan
arkcli usage balance --type plan --product coding-plan
arkcli usage balance --type plan --all
```

## Parameter allowlist

`--type` is required. Other flags are grouped by type. Passing a flag for the wrong type returns `ErrValidation` instead of being silently ignored, so users are not led to believe that an ineffective filter was applied.

| `--type` | Allowed extra flags |
|---|---|
| `free-quota` | `--model` `--modality` `--page-number` `--page-size` |
| `media-asset` | `--project` |
| `plan` | `--product` `--all` `--seat` |

### `--type free-quota`

| Parameter | Type | Description |
|---|---|---|
| `--model` | string | One foundation model; omitted lists all activated models |
| `--modality` | string | Backend `FoundationModelDomain` enum: `LLM` / `ComputerVision` / `Audio` / `Embedding` / `Router` (**aliases are not accepted**) |
| `--page-number` | int | 1-based page number |
| `--page-size` | int | Default 50 |

### `--type media-asset`

| Parameter | Type | Description |
|---|---|---|
| `--project` | string | Project name; defaults to the current `profile.ProjectName` |

### `--type plan`

| Parameter | Type | Description |
|---|---|---|
| `--product` | string | `coding-plan` / `coding-plan-team` |
| `--all` | bool | Query all products even if unsubscribed; mutually exclusive with `--product` |
| `--seat` | string | Team only; filter one seat |

## Return value

### `--type free-quota`

```json
{
  "page_number": 1,
  "page_size": 50,
  "total_count": 7,
  "items": [
    {
      "model": "dola-seed-2-1-turbo",
      "display_name": "Dola-Seed-2.1-turbo",
      "vendor": "Dola",
      "state": "Available",
      "is_overdue": false,
      "free_usage": { "total": 500000, "consumed": 12345, "remaining": 487655 },
      "resource_packs": [
        {
          "type": "FreeInference",
          "total": 500000, "consumed": 12345, "remaining": 487655,
          "reclaimed": 0, "sync_time": "2026-06-04T12:00:00Z"
        }
      ]
    }
  ]
}
```

**Key definitions**:

- `free_usage.remaining = total - consumed` is already calculated; do not have the Agent recalculate it.
- Model rows with both `free_usage` and `resource_packs` empty are omitted. `pricing models` provides the complete model table.
- `resource_packs[].type` currently has only two BytePlus values: `FreeInference` (free inference) / `DataPermission` (data permission). There is no fine-tune resource pack. `Finetune` / `LoraFinetune` are unit-pricing types in `pricing models` under `ChargeItems.Type`, not balance dimensions.
- A resource pack's `remaining` may be negative after expiration or downgrade when `consumed > total`; do not clamp it.

### `--type media-asset`

```json
{
  "tier": "advanced_monthly",
  "assets":      { "total": 100000, "used": 23456, "remaining": 76544, "percent": 23.456 },
  "asset_groups":{ "total": 100,    "used": 12,    "remaining": 88,    "percent": 12.0 },
  "shared_pool": { "total": 1000000,"used": 0,     "remaining": 1000000,"percent": 0 },
  "write_qpm": 600,
  "capabilities": { "liveness_writable": true, "aigc_readable": true },
  "projects": [
    { "project_name": "default", "used": 23456, "allocated": false, "allocation": 0 }
  ],
  "updated_at_ms": 1717488000123
}
```

**Key definitions**:

- `tier`: `free` / `advanced` / `premium` / `advanced_monthly` / `premium_monthly`; monthly and annual subscriptions share these display values. An empty value means that the account has no media-asset subscription and the backend returned a zero-capacity free tier.
- **`used > total` is valid** after an expired subscription is downgraded: `tier` becomes `free`, while `used` retains its previous value. `percent` may exceed 100 and `remaining` may be negative; do not clamp either.
- `shared_pool` is omitted for accounts without the shared pool enabled because all its values are zero.
- `capabilities` lists only enabled capabilities; an empty map means that all capabilities are disabled.
- `projects[]` contains data only for enterprise accounts with project-level quota allocation enabled; it is normally empty for personal accounts.

### `--type plan`

```json
{
  "viewer": {
    "auth_method": "sso",
    "user_id": "12345",
    "user_name": "alice",
    "is_root": false,
    "profile": "my-profile",
    "tenant": "byteplus",
    "region": "ap-southeast-1"
  },
  "items": [
    {
      "product": "coding-plan",
      "edition": "personal",
      "error": "not subscribed"
    }
  ]
}
```

**Key definitions**:

- **Top-level `viewer`**: identity summary for distinguishing the sub-user's own view, the root-account view, and another identity's view across SSO / AK/SK / API Key authentication. See the [`usage plan` viewer section](arkcli-usage-plan.md#viewer-fields-identity-summary).
- **Probes both SKUs by default** (personal × 1 + team × 1), retrieving every plan balance under the account in one operation. Pass `--product=X` to query only one.
- This output is **more concise than `usage plan`**: it removes the `subscribed` / `update_timestamp` metadata fields but retains `error` to distinguish "not subscribed", `AccessDenied`, and "no seat bound".
- `error: "not subscribed"` is a balance-layer hint. The raw backend response is `Subscribed=false` with empty periods, which is not sufficiently clear for users.
- **Team plans automatically resolve the caller's seat**: when `--seat` is omitted, the service calls `GetSeatInfo` to find the bound seat. If the plan is subscribed but no seat is assigned, it returns `error: "no seat bound to caller for ..."` to distinguish this case from "not subscribed".
- The `coding-plan` backend returns only `percent`; `used` / `total` are omitted through JSON `omitempty`.
- `periods[].reset_at` uses the same schema as `usage plan` and is returned in **RFC3339 Beijing time (UTC+08:00)**. It is omitted when the window contains no data.

## Agent decisions

| User asks | Command |
|---|---|
| Which models still have free quota | `arkcli usage balance --type free-quota` |
| How many tokens model X has left | `arkcli usage balance --type free-quota --model X` |
| Media capacity remaining/full | `arkcli usage balance --type media-asset` |
| Plan remaining/reset date | `arkcli usage balance --type plan` |
| Specific product balance | `arkcli usage balance --type plan --product coding-plan-team` |
| Admin view of another seat | `arkcli usage balance --type plan --product coding-plan-team --seat <id>` |

## Common errors

| Error | Cause | Handling |
|---|---|---|
| `--type is required` | `--type` was omitted | Choose one of `free-quota` / `media-asset` / `plan` |
| `--product is not valid for --type free-quota` | A flag was passed across types | Check that each flag matches `--type` in the allowlist above |
| Empty `tier` + all values zero | Account has no media-asset subscription | The backend returned a valid zero-capacity free tier; subscribe through the BytePlus ModelArk console if needed |
| `foundation model "X" not found in pricing list` (exit=2) | `--model X` is misspelled or absent from the pricing list | Check the spelling, or omit `--model` to list supported names |
| `free-quota` returns `items: []` | `--modality` used an alias such as `text` / `video` instead of a backend enum; this path applies when `--model` is omitted | Pass `LLM` / `ComputerVision` instead |
| Plan row has `error: "no seat bound"` | Team plan is subscribed but the caller has no assigned seat | Contact an admin to assign a seat, or explicitly pass `--seat <id>` |

## Relation to other commands

- **Unit price** → [`arkcli pricing models`](../../arkcli-pricing/references/arkcli-pricing-models.md)
- **Complete subscription quota snapshot with `subscribed` and timestamps** → [`usage plan`](arkcli-usage-plan.md)

## References

- [arkcli-usage](../SKILL.md) — Usage overview
- [arkcli-shared](../../arkcli-shared/SKILL.md) — Authentication/global parameters
- [arkcli-pricing](../../arkcli-pricing/SKILL.md) — Pricing catalog view
