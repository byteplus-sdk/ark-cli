---
name: arkcli-onboard
version: 0.2.0
description: "Guide a user through production integration of a BytePlus Ark model: authenticate, verify the account gate, select a deployable model, and reuse a matching running inference Endpoint or create one safely. Trigger for intent-level requests such as integrating a model into an application or service when the user has not already asked for a specific deployment command."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli --help"
---

# BytePlus Ark production onboarding

<!-- Future BytePlus code-example support:
Restore optional code-example generation after Endpoint creation and include
the generated example location in the completion contract.
-->

Before using this Skill, read
[`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md). This is an
orchestration Skill: it owns the order and branching between capabilities, not
the private flags or response schemas of each delegated command.

## When To Trigger

- "Integrate a BytePlus Ark model into my application."
- "Help my backend call this model in production."
- "I selected a model; help me obtain a reusable Endpoint"
- Similar intent-level production-integration requests that do not already
  specify an exact deployment command.

## When NOT To Trigger

- The user explicitly asks to deploy or create an Endpoint: route directly to
  [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md).
- The user only wants a one-off trial, answer, image, or video: route to
  [`../arkcli-chat/SKILL.md`](../arkcli-chat/SKILL.md) or
  [`../arkcli-gen/SKILL.md`](../arkcli-gen/SKILL.md).
- The owning model Skill classifies the target as discoverable but not
  deployable: stop after model discovery and explain the Ark CLI capability
  boundary.

## Execution modes

### Normal workflow

Use remote read operations as needed, then ask for confirmation at the single
write boundary.

### Help-only or no-remote workflow

- If the user supplies a model and explicitly forbids remote queries, do not
  run model search or Endpoint listing.
- To validate command selection without remote calls, use help only:

  ```bash
  arkcli models --help
  arkcli infer endpoint list --mine --help
  arkcli +deploy --help
  ```

- Help-only validation must not log in, activate a model, create an Endpoint,
  or query account resources.

## Agent execution order

Use these steps in order:

1. Authenticate and apply the BytePlus account gate.
2. Select and confirm a deployable model.
3. Reuse an exact matching available Endpoint when one exists.
4. Otherwise, delegate guarded Endpoint creation to `arkcli-deploy`.
<!-- 5. Optionally generate code examples when BytePlus supports them. -->
5. Return the integration result.

```text
Step 0  Authenticate and apply the BytePlus account gate
          |
          +--> arkcli auth status --format json
          |      |
          |      +--> session missing or expired: arkcli auth login, then resume
          |      +--> byteplus_sso.identity.verified == false:
          |      |      stop at https://console.byteplus.com/user/basics/
          |      +--> verified == true: continue
          |      +--> verified missing: continue without claiming a result
          |
Step 1  Select and confirm a deployable model
          |
          +--> delegate search/get to arkcli-models
          +--> no model supplied: present candidates and ask the user to choose
          +--> discoverable-only or unsupported target: stop
          |
Step 2  Look for a reusable inference Endpoint
          |
          +--> arkcli infer endpoint list --mine --page-all
          +--> match the exact model/version and an available status
                 |
                 +--> one match: reuse it
                 +--> multiple matches: ask the user which Endpoint to use
                 +--> no match: continue to Step 3
          |
Step 3  Create an Endpoint
          |
          +--> delegate to arkcli-deploy
          +--> use read-only checks, state that +deploy has no --dry-run, then obtain explicit confirmation
          +--> this is the only write in the onboarding workflow
          |
Step 4  Return the production integration result
                 endpoint ID + model ID + reused/created status
```

## Endpoint reuse rules

Reuse an Endpoint only when all of these are true:

- it belongs to the current SSO sub-user result set from `--mine`;
- it is bound to the exact target model and version;
- its status is available for inference, such as `Running`.

An Endpoint for another model is not a match. Do not silently choose among
multiple matches. Listing all pages is required so a reusable Endpoint is not
missed because of pagination.

## Write and verification guards

- This production workflow can lead to a billable write, so Step 0 applies the
  combined BytePlus account-opening and payment-verification gate before model
  activation or deployment.
- `byteplus_sso.identity.verified == false` does not reveal which underlying
  check is incomplete. Use the single BytePlus account settings URL and do not
  guess.
- A missing `verified` field is unknown, not success or failure. Continue and
  preserve any later structured backend error and request ID.
- Step 3 must follow `arkcli-deploy` safety rules. Do not bypass confirmation
  through an API Explorer action or an undocumented environment variable.
- Do not create a second Endpoint when Step 2 found a valid reusable one.

## Completion contract

The final response must return to the user's original integration goal and
include:

- the selected model ID;
- the Endpoint ID;
- whether the Endpoint was reused or created;
- any remaining user action, without inventing a console URL.

## Guard Checklist

- Read `arkcli-shared` before starting.
- Keep authentication and verification as intermediate gates, not final
  outcomes.
- Delegate command-specific flags to the owning Skill.
- Use `--mine --page-all` for reusable Endpoint discovery.
- Confirm the exact model before deployment.
- `+deploy` has no Client Preview. Use read-only checks and explicit confirmation before the single write.
- Stop on a known failed BytePlus account gate.
- Do not turn a one-off trial into an Endpoint deployment.

## References

- [`../arkcli-auth/references/realname-gate.md`](../arkcli-auth/references/realname-gate.md)
  - combined BytePlus account-opening and payment-verification contract
- [`../arkcli-models/SKILL.md`](../arkcli-models/SKILL.md) - model selection
- [`../arkcli-infer-endpoint/SKILL.md`](../arkcli-infer-endpoint/SKILL.md) -
  Endpoint listing and exact-match semantics
- [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md) - guarded Endpoint creation

- [`references/evals.md`](references/evals.md) - routing and workflow
  evaluations
