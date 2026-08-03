# plans buy

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

> **⚠️ BytePlus payment flow:** BytePlus does not yet support balance-based auto-pay via arkcli. Adding `--yes` **does not place an order or charge the account**; instead the command returns a `payment_via_web` payload with a `subscribe_url` pointing to the BytePlus console subscription page. The user must open that URL in a browser and complete the purchase on the web. **Do not choose for the user**: the user must explicitly provide `--plan`, `--type`, `--duration`, and `--quantity` for team plans.

> **🔒 Review gate (mandatory process; the Agent must comply):** Without `--yes`, this command **does not place an order** and returns price + `subscribe_url`. The Agent **must**:
> 1. Display `total_amount_usd` and `original_amount_usd` to the user for price transparency.
> 2. Then present the `subscribe_url` and ask the user to complete the purchase in the browser. **Do not rerun the command with `--yes`** expecting an in-CLI charge — BytePlus does not support that path; both `--yes` and the gate resolve to the same subscribe URL.
>
> The BytePlus subscription page handles agreement presentation and signing with the correct BytePlus legal documents on the web; the CLI does **not** return an `agreements` array (see Return value below).

## Three invocation forms

| Form | Behavior | When to use |
|---|---|---|
| Neither `--yes` nor `--estimate` | **Review gate**: returns **price** (USD) + `subscribe_url` + `next_step`; internally uses EstimatePrice and **does not place an order** | Default guidance step; the Agent obtains "price + web URL" in one call |
| `--estimate` (with or without `--yes`) | **Online price inquiry**: calls EstimatePrice, returns the price, and **does not place an order** | Overlaps with the gate; retained as a convenient scripting path for "price only" |
| `--yes` without `--estimate` | **Web redirect** (BytePlus): returns `payment_via_web` payload with `subscribe_url`; **no order-side API is called**, no order or seat is created | The user has confirmed the price; direct them to the web console to complete the purchase |

> **The gate returns pricing + subscribe URL together**. One call returns `total_amount_usd` + `original_amount_usd` + `subscribe_url`; show these to the user together and then hand off the URL.

## Commands

```bash
# Step 1: Guidance (without --yes) — obtain the price + subscribe_url
arkcli plans buy --plan coding-plan --type lite --duration 1

# Step 2 (optional): Price inquiry
arkcli plans buy --plan coding-plan --type lite --duration 1 --estimate

# Step 3: After showing the user the price, hand off to the web page via subscribe_url
arkcli plans buy --plan coding-plan --type lite --duration 1 --yes

# Example for a team-edition tier
arkcli plans buy --plan coding-plan-team --type pro --quantity 10 --yes
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `--plan` | Yes | string | `coding-plan` / `coding-plan-team` |
| `--type` | Yes | string | Tier: coding-plan* accepts `lite/pro` |
| `--duration` | No | int | Subscription duration (months), 1–12, default 1 |
| `--quantity` | Required for team | int | Number of seats (≥1); ignored when passed for personal edition |
| `--yes` | No | bool | On BytePlus, returns the `payment_via_web` payload directly (no charge; equivalent to the gate for the subscribe_url purpose) |

Compatibility between `--plan` and `--type`:

| Plan family | Valid tiers |
|---|---|
| coding-plan / coding-plan-team | lite, pro |

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

Amount fields are always USD on BytePlus (`total_amount_usd` / `original_amount_usd`); `total_amount_cny` / `original_amount_cny` will never appear. There is intentionally **no `agreements` array** — the BytePlus subscription page handles agreement presentation and signing on the web with the correct BytePlus legal documents.

## Return value (`--yes` / web redirect)

After adding `--yes`, the CLI does **not** place an order or invoke any Create* API. Instead it returns:

```json
{
  "status": "payment_via_web",
  "plan": "coding-plan-team",
  "tier": "pro",
  "duration": 2,
  "quantity": 5,
  "subscribe_url": "https://console.byteplus.com/ark/region:ark+ap-southeast-1/subscription/coding-plan-enterprise",
  "message": "BytePlus balance-based auto-pay is not supported by arkcli yet. Open the subscribe_url above in the browser to complete the purchase on the BytePlus console.",
  "next_step": "Ask the user to open <subscribe_url> in the browser and finish the purchase on the BytePlus subscription page. Do not rerun this command with --yes."
}
```

`quantity` is populated only for team plans. For team plans **no seats are created**: the CLI does not call `CreateSeatInfo` before redirecting, so there are no dangling seats to clean up.

## Failure semantics

BytePlus arkcli does not place orders, so the `payment_failed` envelope (which the Volc build emits when auto-pay fails after `CreateSubscribeTrade`) **does not exist here**. Failure modes are:

- **Validation error** (e.g. `--quantity is required for team plans`): normal error, exit code = 1. Fix the arguments and retry.
- **Auth / preflight error** (missing `arkcli auth login`, etc.): normal error, exit code = 1. Log in and retry. See [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md).
- **Payment failure on the web**: after the user opens `subscribe_url`, if BytePlus web rejects the payment (e.g. account not opened, real-name pending, payment method missing), the console will surface the failure and the appropriate settings page. This is out of arkcli's control.

## Common errors

| Error | Cause | Handling |
|---|---|---|
| `--plan must be one of ...` | Plan key is misspelled | Strictly use `coding-plan` or `coding-plan-team` |
| `--type %q not allowed for --plan %q` | Tier is incompatible with plan | See the table above |
| `--quantity is required for team plans` | Team quantity omitted | Add `--quantity N` |
| `--duration must be 1..12` | Out of range | Use 1–12 |

## Notes

- **No in-CLI charge**: BytePlus arkcli never invokes any Create-trade API. The purchase always happens on the web console.
- **Use `--estimate` for pricing scripts** if you only need a price quote and want to skip the agreement + subscribe URL machinery.
- After the user completes the purchase on the web console for a team plan, seats appear with `BillingStatus=Running` but are **not bound to sub-users**. Next use [`plans team seat-assign`](arkcli-plans-team-seat-assign.md) to assign them.
- Rerunning `plans buy --yes` is safe on BytePlus: each call is a pure web-redirect stub that produces the same `subscribe_url`. It never places an order.

## References

- [arkcli-plans](../SKILL.md) -- Skill overview
- [`plans renew`](arkcli-plans-renew.md) -- Renew an existing plan
- [`plans get`](arkcli-plans-get.md) -- Query holdings after ordering
- [`plans team seat-assign`](arkcli-plans-team-seat-assign.md) -- Bind sub-users after a team order
- [arkcli-shared](../../arkcli-shared/SKILL.md)
