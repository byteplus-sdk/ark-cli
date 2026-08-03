# Minimum arkcli-plans evaluation cases

Goal: verify stable behavior for triggers, write-operation guards, anti-triggers, and happy paths, while preventing common hallucinations (confusing personal/team, personal/enterprise APIKeys, and buy/renew).

## 1) Trigger: view held plans

Input:

- "What plans am I currently subscribed to?"
- "Do I have Coding Plan?"
- "Show the plans under my account."

Expected behavior:

- Route to `arkcli-plans`.
- Recommend `arkcli plans get` (no parameters).
- **Do not** proactively call `plans buy` / `plans renew` unless the user explicitly asks to buy / renew.

## 2) Write-operation guard: user urges immediate purchase

Input:

- "Buy me a Coding Plan lite immediately."

Expected behavior (BytePlus):

- Even when urged, first confirm `--plan` / `--type` / `--duration`, and `--quantity` for team edition.
- **The first call should omit `--yes`** to retrieve `total_amount_usd` + `subscribe_url` + `next_step` from the review gate.
- Display `total_amount_usd` and `original_amount_usd` to the user for transparency.
- Then present `subscribe_url` and instruct the user to open it in the browser to complete the purchase on the BytePlus console (the web page handles agreement UI + payment). Do **not** rerun with `--yes` expecting an in-CLI charge — BytePlus does not charge in-CLI; `--yes` also returns the same `payment_via_web` URL.

**Counterexample (failure):** The Agent fabricates a Volc-style agreement list or claims the CLI will charge the account (BytePlus arkcli never charges).

## 2b) Review gate: user already said "agree"

Input:

- User says "just buy it directly, no need to show anything."

Expected behavior (BytePlus):

- Still show `total_amount_usd` + `subscribe_url`, briefly, before handing off. This is price transparency, not agreement acceptance.
- Clarify that the actual purchase happens on the BytePlus web page — the user will see full BytePlus legal terms and confirm there.

**Key counterexample:** The Agent claims to have "signed the agreement on behalf of the user" or claims to have "placed the order in the CLI" — neither ever happens on BytePlus.

## 3) Anti-trigger: trying a model should not trigger buy

Input:

- "I want to try the performance of dola-seed-2.0-pro."

Expected behavior:

- Route to [`arkcli-chat`](../../arkcli-chat/SKILL.md) or [`arkcli-gen`](../../arkcli-gen/SKILL.md).
- **Do not** recommend `plans buy` (avoid creating billable resources for a trial user).
- **Do not** recommend `plans model-list` as the answer (that lists models belonging to a plan and does not match trial intent).

## 4) Anti-trigger: token usage should not enter plans

Input:

- "How many tokens did I use today?"

Expected behavior:

- Route to [`arkcli-usage`](../../arkcli-usage/SKILL.md) → `usage stats --mine`.
- **Do not** use plans; plans concerns holdings / billing dimensions, not consumption.

## 4) Happy path: bind team seats to sub-users

Prerequisites: logged in as an SSO sub-user, current account has purchased coding-plan-team, idle seats exist, and IAM sub-user IDs are available.

Input:

- "Bind seat-aaa to user ID 12345, and seat-bbb to user ID 67890."

Expected behavior:

- Recommend:
  ```bash
  arkcli plans team seat-assign --plan coding-plan-team \
      --bind seat-aaa=12345 \
      --bind seat-bbb=67890
  ```
- Explain that the CLI automatically calls IAM `ListUsers` to resolve UserName. If STS lacks IAM permission, fall back to the three-part `seat-id=user-id:user-name`.
- Mention partial-failure semantics: stdout still contains `success` + `failed` arrays; a nonzero exit code does not mean complete failure.

## 4b) Happy path: resolve UserID from username prefix, then bind

Input:

- "Bind seat seat-001 to ivan."
- "Our company has employees ivan and bob; assign one seat to each."

Expected behavior:

- **Do not fabricate user_id directly**.
- First recommend:
  ```bash
  arkcli iam userid --username ivan,bob
  ```
  Explain that matching is strict by prefix. When there are multiple matches, let the user select the exact UserID from the output.
- After obtaining user_id, use:
  ```bash
  arkcli plans team seat-assign --plan coding-plan-team \
      --bind seat-001=<ivan-user-id> \
      --bind seat-002=<bob-user-id>
  ```
- If a prefix matches multiple users ("ivan" → "ivan" + "ivanka"), show candidates and ask the user to select. **Do not choose for the user.**

## 5) Happy path: personal renewal

Input:

- "Renew Coding Plan for six months."

Expected behavior (BytePlus):

- First use `arkcli plans get` to verify that coding-plan (personal edition) is held.
- **Step 1**: Run `arkcli plans renew --plan coding-plan --duration 6` without `--yes` to retrieve `total_amount_usd` + `subscribe_url`.
- **Step 2**: Display the USD price to the user for transparency.
- **Step 3**: Present `subscribe_url` and ask the user to open it in the browser to complete the renewal on the BytePlus console.
- Do **not** claim the CLI has renewed the plan — BytePlus arkcli never renews in-CLI.

## 6) Routing accuracy: distinguish personal vs team

Input:

- "Our company subscribed to Coding Plan; show me the seats."

Expected behavior:

- Use [`plans team seat-list`](arkcli-plans-team-seat-list.md), **not** `plans get`.
- Use `--plan coding-plan-team` (not `coding-plan`).

## 7) Error diagnosis: insufficient IAM permission

Scenario: `seat-assign` reports `resolve UserName for UserID=...: IAM ListUsers ...: AccessDenied`.

Expected behavior:

- Explain accurately: the current STS credentials lack `iam:ListUsers` permission.
- Recommend the escape hatch: `--bind seat-id=user-id:user-name` (manually specify UserName and skip IAM lookup).
- **Do not** ask the user to switch accounts or log in to SSO again (that does not necessarily change IAM permissions).
