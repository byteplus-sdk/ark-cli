# plans get

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

> **Scope limitation:** `plans get` answers only "which plans do I **hold** + status (Effective/Running/...)" and **does not answer "how much has been used / how much remains / when it resets / breakdown by model"**. For quota queries, use:
> - `arkcli usage plan` — Full version with used / total / percent / reset_at + multiple periods (5h/daily/weekly/monthly)
> - `arkcli usage balance --type plan` — Simplified version (same data source)
>
> See [`../../arkcli-usage/SKILL.md`](../../arkcli-usage/SKILL.md).

Return all plans **actually held** by the current account in one call: an aggregation of Coding Plan personal + team editions. This is a read operation with no side effects.

## Command

```bash
arkcli plans get
```

## Parameters

No parameters. Both plan paths (`coding-plan` / `coding-plan-team`) are fetched concurrently.

## Return value

```json
{
  "plans": [
    {
      "key": "coding-plan",
      "name": "Coding Plan",
      "scope": "personal",
      "tier": "lite",
      "status": "Effective"
    },
    {
      "key": "coding-plan-team",
      "name": "Coding Plan Team",
      "scope": "team",
      "tier": "lite",
      "status": "Running",
      "seat_id": "seat-..."
    }
  ]
}
```

| Field | Appears when | Meaning |
|------|---------|------|
| `key` | always | Stable identifier: coding-plan / coding-plan-team |
| `scope` | always | personal / team |
| `tier` | When held | Personal: lite/pro (coding); team: tier of the current user's seat |
| `status` | When held | Personal: Effective / Pending / Expired, and others; team: Running |
| `seat_id` | Team only | Current user's seat ID under this Scene (team edition only) |
| `error` | API call fails | Diagnostic information passed through from the API; mutually exclusive with tier/status |

**Plans not held / historical Reclaimed subscriptions / team plans without a seat** are filtered and do not appear in the array.

## Behavior details

- Internally fetches `ListSubscribeTrade` (personal) + `GetSeatInfo` (team) concurrently.
- Failure of one plan does not block other plans: the failed item remains in the array with an `error` field.
- The team edition recognizes only valid seats with `BillingStatus=Running`; historical seats (Pending/Expired/Reclaimed) are filtered.

## Common errors

| Error | Cause | Handling |
|------|------|------|
| `plans get: ark: API error: AuthFailure` | Not authenticated / token expired | `arkcli auth login` |
| Output `{"plans":[]}` | The current account has no plans | Place an order with `plans buy` or check whether the account was switched |

## Notes

- This is the **only plans subcommand requiring no parameters**. When the user asks "What plans do I have / what have I subscribed to?", invoke it directly.
- **To view usage / quota (used / total / percent / reset date)**: this command **does not return** those fields. Route to `arkcli usage plan` / `arkcli usage balance --type plan` / `arkcli usage plan-details` (see the scope limitation at the top).
- It does not list other users' plans. The view always belongs to the current SSO identity.
- The team edition shows only seats **held by the current user**. To view all team seats, use [`plans team seat-list`](arkcli-plans-team-seat-list.md) (basic information) or `arkcli usage seats --with-usage` (including seat usage).

## References

- [arkcli-plans](../SKILL.md) -- Skill overview
- [`plans model-list`](arkcli-plans-model-list.md) -- See which models can be called under held plans
- [`plans buy`](arkcli-plans-buy.md) / [`plans renew`](arkcli-plans-renew.md) -- Purchase / renew
- [arkcli-usage](../../arkcli-usage/SKILL.md) -- Plan usage / quota (all fields absent from this command are here)
- [arkcli-shared](../../arkcli-shared/SKILL.md)
