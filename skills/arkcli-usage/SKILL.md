---
name: arkcli-usage
version: 1.3.3
description: "ARK usage queries: `usage stats` (tokens/requests, 5–30 minute delay), `usage plan` / `usage balance --type plan` (plan quota snapshots), `usage balance` (free quota/media asset/plan balances), and `usage seats --with-usage` (team usage by seat). Trigger for usage, consumed amount, remaining quota, plan percentage, in-plan/out-of-plan, or per-seat consumption. Seat listing/binding/assignment is management and routes to arkcli-plans; this skill answers usage only. Verb routing: use/consume/how much → here; list/bind/assign → arkcli-plans. Anti-trigger: TTS/ASR/voice model usage is unsupported."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli usage stats --help"
---

# arkcli usage

**CRITICAL — Before starting, you MUST use Read to open [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md), which contains authentication gates, configuration troubleshooting, and shared safety rules.**
**CRITICAL — Before executing `usage stats`, use Read to open [`references/arkcli-usage-stats.md`](references/arkcli-usage-stats.md); do not invoke blindly.**
**CRITICAL — Before executing `usage plan`, use Read to open [`references/arkcli-usage-plan.md`](references/arkcli-usage-plan.md); do not invoke blindly.**
**CRITICAL — Before executing `usage balance`, use Read to open [`references/arkcli-usage-balance.md`](references/arkcli-usage-balance.md); do not invoke blindly.**
**CRITICAL — Before executing `usage seats`, use Read to open [`references/arkcli-usage-seats.md`](references/arkcli-usage-seats.md); do not invoke blindly.**

> **Data freshness (Must read first): `usage stats` uses an upstream BFF aggregation pipeline and has a 5–30 minute delay. It is intended for daily/aggregate analysis.** Do not use it for real-time budget monitoring, rate limiting, or alerts requiring second-level precision. For those scenarios, read per-request `.usage` returned by inference commands such as `+chat` / `+gen` (zero delay; see [arkcli-chat](../arkcli-chat/SKILL.md) and [arkcli-gen](../arkcli-gen/SKILL.md)).

> **Agent Plan boundary**: BytePlus does not offer Agent Plan (`agent-plan` / `agent-plan-team`). When a user asks about **Agent Plan** quota, remaining, reset date, or per-seat usage, respond with "Agent Plan is not offered on BytePlus" and stop — do not silently answer with Coding Plan numbers and do not attempt any Agent Plan flag combination as a probe.

> **Voice model boundary**: TTS / ASR / dubbing / reading / podcast / voice / real-time voice interaction are currently unsupported in arkcli. Do not answer their usage with `usage stats --model` or plan/balance commands; state directly that arkcli does not support querying voice model pricing or usage.

> **Difference among stats / plan / balance:**
> - `stats`: token-billed inference usage (any identity, 5–30 minute delay).
> - `plan`: subscription quota snapshot (CodingPlan personal + team: percentage used / reset date).
> - `balance`: "how much X remains" from a balance perspective (`--type free-quota / media-asset / plan`).
> - `seats`: admin view of team-seat **usage** (per-seat usage / plan percentage; **usage only**). Seat listing/binding/assignment uses [`arkcli-plans`](../arkcli-plans/SKILL.md): `plans team seat-list / seat-assign`.
>
> "How much plan quota remains?" → `plan` or concise `balance --type plan`; "How many tokens did I use today?" → `stats`; "How much free quota/media capacity remains?" → `balance`; "**How much did each seat use / team-seat usage distribution?**" → `seats --with-usage`. Pure management ("who is bound / list seats / assign seats") → `arkcli-plans`.

## Step 0: choose the metric, then verify the identity

- Explicit Coding Plan quota questions use `usage plan --product coding-plan|coding-plan-team`. Subscription ownership belongs to the account/user, not the active profile type; a platform profile does not prove there is no subscription.
- Platform Token/request usage uses `usage stats --start <YYYY-MM-DD>`; personal scope must retain `--mine`, including when adding grouping. Only query both buckets when the user asks for both; label them separately and do not sum quota percentages and Tokens.
- Coding Plan has quota snapshots, not per-model/Harness or Endpoint/API Key attribution. State this boundary; do not append platform stats as a substitute. Agent Plan is not offered on BytePlus.
- If "my usage" is ambiguous, clarify the metric from the current context before querying. Do not switch Profile, Key, Project or account scope to fill gaps. Reuse known identity context; when a missing summary is needed, use `arkcli auth whoami --format json`. Ordinary admission does not run `profile show/list/keys list`, which can synchronize and write back keys.
- Stats require explicit `--start`; derive dates from the request and disclose a today-only default. Split long ranges according to the reference instead of silently shortening them.
- Distinguish failed queries, successful empty records and nonempty zero totals. Check time range, aggregation delay, metric, identity and filters before concluding no usage. Recent missing records do not prove no call occurred.
- Report metric, identity/scope, dates and freshness. Platform attribution uses actual `ModelEndpoint` / masked `AuthToken` evidence; do not retrieve plaintext keys merely to identify usage.

## Applicable scenarios

- View inference usage for today or a date range (**accepting 5–30 minute delay**).
- View subscription plan quota consumption / reset time.
- Group stats by model, endpoint, or API Key.
- Analyze token trends and produce reconciliation reports.
- **Not for voice model usage**: for TTS / ASR / voice-model usage, remaining quota, or calls, do not run this skill; state that it is unsupported.

## Business positioning

- This skill handles existing invocation results and consumption; it is an **offline / near-real-time** analysis entry.
- It normally follows:
  - An existing Endpoint whose usage is queried.
  - A model already being called, analyzed by model / API Key.
  - A subscribed plan whose remaining quota is queried.
- It is not an Endpoint creation entry or a model trial entry.
- **It is not a data source for real-time budget/rate-limit/alert controls**; use per-request `.usage`.

## Quick decisions

- "How much plan quota remains / percentage used / reset date / how many plans":
  - `arkcli usage plan` (probes all two SKUs; only subscribed buckets issue usage requests).
  - `arkcli usage plan --all`: query both buckets without subscription probing (diagnostics/Excel).
  - `arkcli usage plan --product=X`: skip probing and query one.
    - **This is the same data as `balance --type plan`**. `usage plan` provides the complete output (`subscribed` / `updated_at`), while `balance --type plan` provides a concise projection without metadata. Prefer `usage plan`; use `balance --type plan` only when reconciling plan balances together with free quota or media-asset capacity.
- "My usage / tokens today / endpoint consumption":
  - First choose the requested metric using Step 0. An explicit platform/Endpoint request does not need a plan query first.
  - If the user names a voice model/TTS/ASR → **do not query usage**. Currently arkcli doesn't support querying the usage of audio models.
  - **BytePlus:** first run `arkcli usage stats --start <YYYY-MM-DD> --mine` (endpoint dimension).
    - `data_count > 0` → use `totals`.
      - `data_count=0` (the user has no Endpoint or has not invoked models through an Endpoint recently) → immediately retry `arkcli usage stats --start <YYYY-MM-DD> --mine --mine-by=apikey`.
    - **⛔ Never degrade to an account-wide query without `--mine`**. Whole-account data is not "my usage".
- "How much did each APIKey use?": `arkcli usage stats --start <YYYY-MM-DD> --mine --mine-by=apikey`; split by `records[].AuthToken`.
- "Remaining free tokens/model free quota/media capacity":
  - `arkcli usage balance --type free-quota`.
  - `arkcli usage balance --type media-asset`.
- "How much team seats used / per-seat consumption / plan percentage by seat" (admin usage view):
  - `arkcli usage seats --product coding-plan-team --with-usage`.
  - Without `--with-usage`, only metadata is returned.
  - Routing: this command answers only usage. "**List seats / inspect seat bindings / assign seats to employees**" routes to [`arkcli-plans`](../arkcli-plans/SKILL.md) and its `plans team seat-list / seat-assign` commands. Verbs "used/consumed/how much" stay here; "bind/list/assign" goes to plans.
- If data looks wrong, check authentication/profile before questioning the data.

## Agent quick execution order

1. Confirm authentication; if uncertain, run `arkcli auth status`.
2. Start with the smallest query, then add date ranges and grouping.
3. For abnormal results, first check profile / region / API Key source.
4. For endpoint usage, preferably confirm endpoint-id first.

## Common fallbacks

- Authentication error: [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).
- Wrong query environment: [`../arkcli-config/SKILL.md`](../arkcli-config/SKILL.md).
- No Endpoint and preparing integration: [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md).
  - Plan **price**, not usage: [`../arkcli-pricing/SKILL.md`](../arkcli-pricing/SKILL.md). `pricing plans` returns catalog pricing, while `usage plan` returns quota snapshots; they are different views.
  - **Settlement amount/bill/money spent**, not tokens: [`../arkcli-billing/SKILL.md`](../arkcli-billing/SKILL.md). Its data source is the BytePlus billing center and is posted at T+1; `usage stats` uses near-real-time BFF inference aggregation.
- Agent Plan quota, remaining balance, or per-seat usage: clearly state that Agent Plan is not supported, take no action, and do not use any Coding Plan commands as a substitute.
- Voice/TTS/ASR usage or quota: state that arkcli does not support it; do not route to pricing/billing.

## Command overview

| Command | Description |
|------|------|
| [`usage stats`](references/arkcli-usage-stats.md) | Inference usage (tokens/requests, daily/hourly aggregation; token-billed inference view) |
| [`usage plan`](references/arkcli-usage-plan.md) | Subscription quota snapshots (CodingPlan; usage, remaining quota, and reset time for 5h / weekly / monthly / session windows). Probes both SKUs by default |
| [`usage balance`](references/arkcli-usage-balance.md) | Unified balance entry through `--type free-quota / media-asset / plan` (model free inference quota / media-asset capacity / plan balance) |
| [`usage seats`](references/arkcli-usage-seats.md) | **Admin view** listing all enterprise plan seats (SeatID / bound sub-user / billing status). Sub-users normally receive `AccessDenied` |

## Natural-language triggers

| User says | Command |
|---|---|
| "remaining quota / balance / how much remains / percentage used / plan quota" | `arkcli usage plan` or `arkcli usage balance --type plan` |
| "remaining free quota / model free quota / free tokens" | `arkcli usage balance --type free-quota` |
| "my usage / tokens today / inference usage / token consumption" | `arkcli usage stats --start <YYYY-MM-DD> --mine` |

## References

- [arkcli-deploy](../arkcli-deploy/SKILL.md) -- Create Endpoint first, then inspect endpoint usage
- [arkcli-pricing](../arkcli-pricing/SKILL.md) -- Catalog pricing from `pricing plans` complements `usage plan` quota snapshots
- [arkcli-shared](../arkcli-shared/SKILL.md) -- Authentication and global parameters
