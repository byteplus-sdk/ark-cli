# plans renew

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

> **⚠️ BytePlus payment flow:** Like `plans buy`, `plans renew` on BytePlus does not perform in-CLI charging. Adding `--yes` **does not renew or charge**; it returns a `payment_via_web` payload with a `subscribe_url` pointing to the BytePlus console subscription page (the same page where new purchases happen — it also handles renewals). The user completes the renewal in the browser.

> **🔒 Review gate (mandatory process, identical to `plans buy`):** Without `--yes`, this command **does not renew** and returns price + `subscribe_url`. The Agent **must** display `total_amount_usd` / `original_amount_usd` to the user for transparency, then present `subscribe_url` for the browser step. **Do not rerun with `--yes`** expecting an in-CLI renewal — BytePlus does not support that path; both `--yes` and the gate resolve to the same subscribe URL. See the [`plans buy` review gate](arkcli-plans-buy.md). The CLI does **not** return an `agreements` array; the BytePlus subscription page handles agreement UI on the web.


## Three invocation forms

Same as `plans buy`:

| Form | Behavior |
|---|---|
| Neither `--yes` nor `--estimate` | Review gate: returns **price** (USD) + `subscribe_url` + `next_step`; internally uses EstimatePrice (IsRenew=true / Scene=RENEW) and does not place an order |
| `--estimate` | Online quote (scripting path for "price only") |
| `--yes` | **Web redirect** (BytePlus): returns `payment_via_web` payload with `subscribe_url`; **no renewal API is called**, nothing is renewed |

> The gate response **resolves and fills the tier**: personal from `ListSubscribeTrade.InfoList[0].BizInfo`; team from `ListSeatInfos` using SeatIDs[0]. The `echo.tier` field lets the Agent confirm with the user, "Do you still want to renew this tier?"

## Commands

```bash
# Step 1: Guidance — obtain price + subscribe_url
arkcli plans renew --plan coding-plan
arkcli plans renew --plan coding-plan-team --duration 3 --seat-ids seat-001,seat-002

# Step 2: Price inquiry (optional)
arkcli plans renew --plan coding-plan --duration 6 --estimate

# Step 3: After showing the user the price, hand off to the web page via subscribe_url
arkcli plans renew --plan coding-plan --yes
arkcli plans renew --plan coding-plan --duration 6 --yes
arkcli plans renew --plan coding-plan-team --duration 3 --seat-ids seat-001,seat-002 --yes
arkcli plans renew --plan coding-plan-team --seat-ids seat-aaa --yes
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `--plan` | Yes | string | `coding-plan` / `coding-plan-team` |
| `--duration` | No | int | Renewal duration (months), 1–12, default 1 |
| `--seat-ids` | Required for team | string | Comma-separated seat IDs to renew (personal edition **does not allow** this argument) |
| `--yes` | No | bool | On BytePlus, returns the `payment_via_web` payload directly (no charge; equivalent to the gate for the subscribe_url purpose) |

> **Key difference**: Unlike `plans buy`, `renew` does not need `--type`; renewal keeps the original tier. It also does not need `--quantity`; the quantity is determined by the number of `--seat-ids`.

## Return value (review gate, default form)

When neither `--yes` nor `--estimate` is passed:

```json
{
  "status": "agreement_required",
  "plan": "coding-plan",
  "tier": "lite",
  "duration": 1,
  "echo": {"plan": "coding-plan", "tier": "lite", "duration": 1},
  "total_amount_usd": 10,
  "original_amount_usd": 10,
  "subscribe_url": "https://console.byteplus.com/ark/region:ark+ap-southeast-1/subscription/coding-plan",
  "confirm_text": "BytePlus balance-based auto-pay is not supported by arkcli yet...",
  "next_step": "Ask the user to open <subscribe_url> in the browser and finish the purchase on the BytePlus subscription page. Do not rerun this command with --yes."
}
```

`tier` / `echo.tier`: the current tier resolved from the existing subscription / seat. The Agent uses it to confirm the renewal tier with the user.

For team edition, `echo.seat_ids` lists the seat IDs being renewed and the payload's top-level `seat_ids` mirrors them; the `subscribe_url` points to `/subscription/coding-plan-enterprise`.

Amount fields are always USD on BytePlus; `total_amount_cny` / `original_amount_cny` will never appear. There is intentionally **no `agreements` array**.

## Return value (`--yes` / web redirect)

After adding `--yes`, the CLI does **not** invoke any renewal API. Instead:

```json
{
  "status": "payment_via_web",
  "plan": "coding-plan-team",
  "duration": 3,
  "seat_ids": ["seat-001", "seat-002"],
  "subscribe_url": "https://console.byteplus.com/ark/region:ark+ap-southeast-1/subscription/coding-plan-enterprise",
  "message": "BytePlus balance-based auto-pay is not supported by arkcli yet. Open the subscribe_url above in the browser to complete the purchase on the BytePlus console.",
  "next_step": "Ask the user to open <subscribe_url> in the browser and finish the purchase on the BytePlus subscription page. Do not rerun this command with --yes."
}
```

`seat_ids` is populated only for team plans.

## Failure semantics

BytePlus arkcli does not renew orders in-CLI, so the `payment_failed` envelope (which the Volc build emits when auto-renew fails) **does not exist here**. Failure modes are:

- **Validation error** (e.g. `--seat-ids is required for team plans`): normal error, exit code = 1. Fix the arguments and retry.
- **Auth / preflight error**: normal error, exit code = 1. Log in and retry. See [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md).
- **Renewal failure on the web**: after the user opens `subscribe_url`, if the BytePlus web console rejects the renewal, the console will surface the failure. This is out of arkcli's control.

## Common errors

| Error | Cause | Handling |
|---|---|---|
| `--plan must be one of ...` | Plan key is misspelled | Strictly use one of two choices |
| `--seat-ids is required for team plans` | Team seat IDs omitted | List the seat IDs to renew |
| `--seat-ids only applies to team plans` | Passed for personal edition | Remove `--seat-ids`; personal edition does not need it |
| `--seat-ids must contain at least one non-empty seat ID` | Empty string / commas only | Check the input |
| `--duration must be 1..12` | Out of range | Use 1–12 |

## Notes

- **No in-CLI renewal**: BytePlus arkcli never invokes any Renew-trade API. The renewal always happens on the web console.
- Team `--seat-ids` are not deduplicated; duplicate IDs are resolved by the server using the final entry.
- Personal renewal **does not require querying InstanceID first**; the agreement-gate path automatically resolves it with `ListSubscribeTrade` for pricing/tier resolution. The user only needs `--plan`.
- Renewal duration is not constrained by existing expiration-time accumulation; `--duration` 1–12 is the per-call limit.
- Rerunning `plans renew --yes` is safe on BytePlus: each call is a pure web-redirect stub returning the same `subscribe_url`. It never charges the account.

## References

- [arkcli-plans](../SKILL.md) -- Skill overview
- [`plans buy`](arkcli-plans-buy.md) -- Same web-redirect semantics
- [`plans get`](arkcli-plans-get.md) -- View expiration time after renewal
- [`plans team seat-list`](arkcli-plans-team-seat-list.md) -- List renewable team seats
- [arkcli-shared](../../arkcli-shared/SKILL.md)
