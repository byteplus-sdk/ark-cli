---
name: arkcli-plans
version: 1.2.0
description: "Manages ARK plans (Coding Plan, personal + team): query holdings, buy, renew, list models, list enterprise seats (`plans team seat-list`), assign seats to employees (`plans team seat-assign`), rotate team-seat API keys (`plans team rotate-apikey`), and inspect seat bindings. Trigger terms: plan / buy / renew / list seats / view seat bindings / who is bound to which seat / assign employee seats / rotate team API key / team-seat admin view. Unbinding is not currently exposed by arkcli; route such requests here only to explain the limitation. Usage questions (remaining quota / percentage used / per-seat usage) go to [arkcli-usage](../arkcli-usage/SKILL.md). Verb routing: list / bind / assign / rotate → here; use / consumption / how much → arkcli-usage."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli plans --help"
---

# arkcli plans

**CRITICAL — Before starting, you MUST use the Read tool to read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md), which contains authentication gates, configuration troubleshooting, and shared safety rules.**
**CRITICAL — Before executing any `plans buy` / `plans renew` / `plans team seat-assign` / `plans team rotate-apikey`, you MUST use the Read tool to read the corresponding `references/*.md`. Do not invoke them blindly.**

## 🔒 Agreement gate (mandatory process for plans buy / plans renew)

`plans buy` / `plans renew` on BytePlus **never place orders in-CLI**. Adding `--yes` does not trigger a charge; it returns a `payment_via_web` payload with a `subscribe_url` pointing to the BytePlus console subscription page, where the user completes the purchase in the browser. **When neither `--yes` nor `--estimate` is passed**, the CLI runs the review gate and returns:

```json
{
  "status": "agreement_required",
  "subscribe_url": "https://console.byteplus.com/ark/region:ark+ap-southeast-1/subscription/coding-plan",
  "total_amount_usd": 10,
  "original_amount_usd": 10,
  "next_step": "Ask the user to open <subscribe_url> in the browser and finish the purchase on the BytePlus subscription page. Do not rerun this command with --yes."
}
```

Note: BytePlus does **not** return the `agreements` array. The web subscription page handles agreement presentation and signing with the correct BytePlus legal documents; the CLI would only have Volc-region agreement text to show, which is wrong for BytePlus users. Amount fields are `total_amount_usd` / `original_amount_usd` (USD).

**The Agent must strictly follow this sequence:**

1. **The first call must omit `--yes`** to obtain price + `subscribe_url`.
2. Display `total_amount_usd` and `original_amount_usd` to the user for transparency.
3. Then present the `subscribe_url` and ask the user to complete the purchase in the browser on the BytePlus console. **Do not rerun the command with `--yes`** expecting an in-CLI charge — both `--yes` and the gate resolve to the same subscribe URL on BytePlus.

**Violating this process means skipping user-facing price transparency.** The web page also enforces its own agreement UI before the user can pay.

`--estimate` is the online price-quote path; it is not Client Preview. It neither places an order nor returns a `subscribe_url` (use the default gate path when you need the URL).

## Business positioning

`arkcli plans` manages ARK plans:
- **Coding Plan** (`coding-plan` / `coding-plan-team`): programming assistance plans divided into tiers (lite/pro).

Plans have two forms: **personal edition** (`personal`) and **enterprise / team edition** (`team`):

| Form | Resource unit | Typical operations |
|---|---|---|
| personal | Individual subscription | View / buy / renew |
| team | Enterprise seat | View / buy / renew / list seats / bind sub-users |

> A personal account has one subscription; a team plan is managed at SeatID granularity, with each seat bound to one sub-user.

> **This skill does not include plan usage / quota queries** ("how much has been used / how much remains / when it resets / breakdown by model"). `plans get` answers only "which plans do I **hold** + status (Effective/Running)", not "how much has been used". The quota view is in [`../arkcli-usage/`](../arkcli-usage/SKILL.md). See the routing table under "Quick decisions".

> **Agent Plan boundary (BytePlus)**: BytePlus does not offer Agent Plan (`agent-plan` / `agent-plan-team`). When asked about Agent Plan, respond with "Agent Plan is not offered on BytePlus" and stop — never fall back to Coding Plan data as a stand-in, never invent an `agent-plan` invocation to "try".

## Applicable scenarios

- View plans held by the current account → `plans get`.
- Place an order for a plan (personal / team) → `plans buy`.
- Renew an existing plan → `plans renew`.
- View the models supported by a plan → `plans model-list`.
- List / filter enterprise seats → `plans team seat-list`.
- Bind an enterprise seat to a sub-user → `plans team seat-assign`.
- Rotate API keys for explicit Coding Plan Team SeatIDs → `plans team rotate-apikey`.

## Quick decisions

- User asks "What plans do I have / what have I subscribed to?": run `arkcli plans get` directly with no arguments (**holdings + status only, no usage**).
- User asks **"How much have I used / how much remains / when does it reset / in-plan vs out-of-plan"**: this is **outside this skill**; route to [`../arkcli-usage/`](../arkcli-usage/SKILL.md):
  - "How much of my plan remains / when does it reset?" → `arkcli usage plan` or `arkcli usage balance --type plan`.
  - "Which model do I use most / in-plan versus out-of-plan ratio?" → `arkcli usage plan-details`.
  - "Team-seat usage" → `arkcli usage seats --product coding-plan-team --with-usage`.
- User asks "Which models does Coding Plan support?": `arkcli plans model-list --plan <plan>`.
- User wants to "buy / renew a plan": first read [references/arkcli-plans-buy.md](references/arkcli-plans-buy.md) / [renew.md](references/arkcli-plans-renew.md). **Explicitly require `--plan`, `--type`, and `--duration`; team plans also require `--quantity`**. Do not choose for the user.
- User wants to "**query seats / inspect team-seat bindings / see who is bound to which seat / list seats / see activated seats / team-seat admin view**" → `plans team seat-list --plan <coding-plan-team>` (**the default entry point for seat management**; the management view lists basic information + bindings, **without usage numbers**. To see each seat's token usage / plan percentage → `arkcli usage seats --with-usage`).
- User wants to "assign a seat to an employee / allocate employee seats" → `plans team seat-assign`; first prepare a `seat-id=user-id` pairing list.
- The user **knows only the employee username**, not the exact UserID: first run `arkcli iam userid --username <prefix>` to resolve `user_id`, then pass it to `seat-assign --bind seat-id=<user_id>`.
- User wants to rotate one or more Coding Plan Team seat API keys → `plans team rotate-apikey --seat-ids <seat-id,...>`; BytePlus requires explicit SeatIDs and never performs an implicit self lookup. Read the rotate reference, warn that every old key becomes invalid immediately, and obtain explicit confirmation before adding `--yes`.

## Agent quick execution order

1. First confirm authentication status: `arkcli auth status`; if missing, use `../arkcli-auth/`.
2. Execute read operations (`get` / `model-list` / `seat-list`) directly; consider the current identity only when the user asks about "mine".
3. For **write operations (`buy` / `renew` / `seat-assign` / `rotate-apikey`)**, always:
   - Read the corresponding reference first.
   - Confirm key fields with the user (plan / type / duration / SeatIDs / UserID pairings).
4. Partial failure (per-item Success/Failed arrays) is a valid response. When the exit code is nonzero, **do not** immediately treat the entire operation as failed; first inspect `success_count` / `failed_count` in stdout.

## Write-operation risk list (must read)

| Command | Risk | Mandatory action |
|---|---|---|
| `plans buy` | Compliance step; **payment happens on the BytePlus web console**, not in CLI | First run without `--yes` through the review gate → show `total_amount_usd` + `subscribe_url` to the user → direct them to `subscribe_url` in the browser (where BytePlus web handles agreement UI). `--yes` also returns `subscribe_url` and does **not** charge the account. See [agreement gate process](#-agreement-gate-mandatory-process-for-plans-buy--plans-renew) |
| `plans renew` | Same as buy: payment happens on the BytePlus web console | Same review gate as buy; team edition requires `--seat-ids`. `--yes` also returns `subscribe_url` without renewing anything in-CLI |
| `plans team seat-assign` | Modifies seat bindings | Explicit `--bind seat-id=user-id`; automatically calls IAM to resolve UserName |
| `plans team rotate-apikey` | **Credential rotation**; old keys are invalidated immediately | Explicit `--seat-ids`; confirm all affected applications are ready for replacement keys before adding `--yes`; never paste returned plaintext keys into chat |

## Command overview

| Command | Type | Description |
|---|---|---|
| [`plans get`](references/arkcli-plans-get.md) | Read | List plans held by the current account (personal + team aggregation) |
| [`plans buy`](references/arkcli-plans-buy.md) | Review + web redirect | Show price + `subscribe_url` so the user can complete the purchase on the BytePlus console (web handles agreement UI) |
| [`plans renew`](references/arkcli-plans-renew.md) | Compliance + web redirect | Same as buy: `subscribe_url` points to the BytePlus console renewal page |
| [`plans model-list`](references/arkcli-plans-model-list.md) | Read | List models supported by a plan + the currently selected ark-latest-model |
| [`plans team seat-list`](references/arkcli-plans-team-seat-list.md) | Read | List enterprise seats + multidimensional filtering |
| [`plans team seat-assign`](references/arkcli-plans-team-seat-assign.md) | Write | Batch-bind enterprise seats to sub-users |
| [`plans team rotate-apikey`](references/arkcli-plans-team-rotate-apikey.md) | Destructive write | Rotate API keys for explicitly selected Coding Plan Team seats |

## Common fallbacks

- Authentication error: route to [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).
- Wrong region / project: route to [`../arkcli-config/SKILL.md`](../arkcli-config/SKILL.md).
- **View plan usage / quota (used / total / percent / reset_at / model breakdown)**: route to [`../arkcli-usage/SKILL.md`](../arkcli-usage/SKILL.md). This skill answers only "what is held", not "how much was used".
- **View billing / settlement amounts**: route to [`../arkcli-billing/SKILL.md`](../arkcli-billing/SKILL.md). `plans` does not return monetary amounts.
- Try a model without formal deployment: route to [`../arkcli-chat/SKILL.md`](../arkcli-chat/SKILL.md) / [`../arkcli-gen/SKILL.md`](../arkcli-gen/SKILL.md).
- Fall back to [`../arkcli-api-explorer/SKILL.md`](../arkcli-api-explorer/SKILL.md) when no product command exists.

## References

- [arkcli-shared](../arkcli-shared/SKILL.md) -- Authentication / global parameters / output rules
- [arkcli-usage](../arkcli-usage/SKILL.md) -- Plan **usage / quota** views (`usage plan` / `usage plan-details` / `usage balance` / `usage seats --with-usage`), complementing this skill's holdings / purchases / seat management
- [arkcli-billing](../arkcli-billing/SKILL.md) -- Plan **settlement amounts** (BytePlus billing-center breakdown, T+1)
- [arkcli-deploy](../arkcli-deploy/SKILL.md) -- Deploy an endpoint with `+deploy` after purchasing a plan
- [arkcli-helper](../arkcli-helper/SKILL.md) -- **Inject / remove** Plan model/provider for an AI Agent
- [Coding Plan Team API-key rotation](references/arkcli-plans-team-rotate-apikey.md) -- Destructive administrator rotation for explicit SeatIDs
