# plans team seat-list

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

List Coding Plan **enterprise-edition** seats, supporting seven filtering dimensions + pagination. Read operation.

## Commands

```bash

# Coding Plan team edition
arkcli plans team seat-list --plan coding-plan-team

# Filter by seat tier
arkcli plans team seat-list --plan coding-plan-team --type lite

# Precisely query specified seats (up to 1,000 SeatIDs)
arkcli plans team seat-list --plan coding-plan-team --seat-ids seat-001,seat-002

# Filter by bound sub-user name
arkcli plans team seat-list --plan coding-plan-team --user-name yinfan.ivan

# View only idle seats (Idle=1)
arkcli plans team seat-list --plan coding-plan-team --seat-status 1

# View only Running billing status
arkcli plans team seat-list --plan coding-plan-team --billing-status 2

# Pagination
arkcli plans team seat-list --plan coding-plan-team --page-number 2 --page-size 50
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `--plan` | Yes | string | `coding-plan-team` (personal-edition plans are rejected) |
| `--type` | No | string | Tier filter: coding-plan-team accepts `lite/pro` |
| `--seat-ids` | No | string | Comma-separated exact filter; **maximum 1,000 per call** (client checks first) |
| `--user-name` | No | string | Filter by the bound sub-user name |
| `--seat-status` | No | int | 1=Idle (unbound) / 2=Active (bound). Other values are rejected |
| `--billing-status` | No | int | 1=Pending / 2=Running / 3=Expired / 4=Reclaimed |
| `--project-name` | No | string | Project name, default `default` |
| `--page-number` | No | int | Page number (≥1) |
| `--page-size` | No | int | Page size (≥1) |

Compatibility between `--plan` and `--type` is the same as `plans buy`.

## Return value

```json
{
  "plan": "coding-plan-team",
  "scene": "",
  "total": 42,
  "seats": [
    {
      "seat_id": "seat-20260608110804-gjrwr",
      "tier": "medium",
      "seat_status": "Active",
      "billing_status": "Running",
      "user_id": "83144215",
      "user_name": "yinfan.ivan",
      "project_name": "default",
      "instance_id": "ins-...",
      "order_time": 1700000000000,
      "expired_time": 1800000000000,
      "create_time": 1690000000000,
      "update_time": 1700000000000
    }
  ]
}
```

| Field | Description |
|------|------|
| `scene` | Empty string for coding-plan-team (the server defaults to coding_plan) |
| `total` | Total number matched by server-side filters (not necessarily the current page size) |
| `seat_status` | "Idle" / "Active" / "Unknown" (numeric enum translated into readable strings) |
| `billing_status` | "Pending" / "Running" / "Expired" / "Reclaimed" / "Unknown" |
| `user_id` / `user_name` | Bound sub-user identifiers (IAM `IdentityId` / `IdentityDetail`) |
| `*_time` | Epoch milliseconds |

For `coding-plan-team`, output `scene` is an empty string. This is expected; the server defaults an empty Scene to coding_plan.

## Notes

- If the user asks "How many seats do I have?" without specifying a plan, first run `arkcli plans get` to see which team edition is held, then run `seat-list` for it.
- **This command lists only basic information** (SeatID / bound identity / billing status, and others). **To view usage for each seat** (5h/weekly/monthly percent + reset_at), use `arkcli usage seats --product <coding-plan-team> --with-usage`. It uses the same `ListSeatInfos` source and additionally joins `ListSeatAFPUsage` / `ListSeatInfoUsages`.
- Numeric enums (`--seat-status` / `--billing-status`) are explained in this reference. **Do not ask the user to consult documentation**; explain their meanings directly.
- `--seat-ids` is not deduplicated; duplicate IDs are resolved by the server using the final entry.
- `BillingStatus` is unfiltered by default (returns all states). The frontend BindSeatDrawer defaults to `[Running, Expired]`; the CLI is more "raw", so add `--billing-status` as needed.

## References

- [arkcli-plans](../SKILL.md) -- Skill overview
- [`plans team seat-assign`](arkcli-plans-team-seat-assign.md) -- Bind sub-users to idle seats
- [`plans get`](arkcli-plans-get.md) -- First see which team plan is held
- [arkcli-usage](../../arkcli-usage/SKILL.md) -- Seat **usage** view (`usage seats --with-usage`), complementing this command's basic list
- [arkcli-shared](../../arkcli-shared/SKILL.md)
