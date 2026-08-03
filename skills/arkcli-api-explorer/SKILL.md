---
name: arkcli-api-explorer
version: 0.2.0
description: "Inspect and invoke Ark CLI registered actions for BytePlus contract validation or low-level fallback. Use only after confirming that no stable product command covers the request, or when the user explicitly asks about an action, operation, registry entry, `unknown action` error, or exact `--params` payload."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli api --help"
---

# BytePlus Raw API Explorer

Before using this Skill:

1. Read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) for the
   BytePlus product boundary, authentication, output, and confirmation rules.
2. Read
   [`references/arkcli-api.md`](references/arkcli-api.md) before constructing
   any `arkcli api` invocation.

Raw API Explorer is the final fallback, not the default command path.

## When To Trigger

- The user explicitly names an action, operation, registry entry, or raw
  request contract.
- `arkcli api ...` returns `unknown action` or rejects `--params`.
- A contributor needs to confirm that a registered operation is visible and
  routed to the expected protocol and target.
- No stable product command covers a specialized data-plane operation.

## When NOT To Trigger

- A stable product command or workflow covers the goal, including model
  discovery, generation, deployment, Endpoint management, usage, profile, or
  billing.
- The problem is missing login, `401`, permissions, profile selection, Region,
  or base URL. Diagnose authentication or configuration first.
- The action contract is unknown. Do not guess a payload merely because an
  action name appears in the registry.

## Command selection order

1. Check the owning product command and Skill.
2. Use `arkcli <domain> --help` or `arkcli +<workflow> --help`.
3. If no product command applies, list the local registry.
4. Confirm the exact BytePlus request contract.
5. Apply authentication, verification, and write guards.
6. Invoke the registered action and extract only the fields the task needs.

## Registry boundary

```bash
arkcli api --list
arkcli api
```

Both commands list the actions compiled into the local binary, sorted by name.
List mode is local and does not require authentication.

A registry entry proves only that the client knows an operation descriptor. It
does not prove that the active BytePlus account, service, Region, or product
contract supports that action. Require a matching BytePlus capability Skill or
an authoritative BytePlus API contract before invocation. Never switch to
another product to make an action succeed.

## Invocation contract

```bash
arkcli api <registered-action> --params '<json-object>'

# Local client preview: validates and displays the descriptor and payload
# without creating a transport or sending a request.
arkcli api <registered-action> --params '<json-object>' --dry-run
```

- `<registered-action>` must exactly match a name from `arkcli api --list`.
- `--params` is an optional JSON object string and defaults to `{}`.
- Field names and casing come from the action request contract; do not infer
  them from the action name.
- `--dry-run` is a local flag of Raw API invoke mode. It emits a
  `preview.v1` plan with a redacted `steps[0].payload` and does not contact
  BytePlus services.
- `arkcli api --list --dry-run` is invalid because list mode is already a
  local registry read and has no remote request to preview.
- Prefer `--format json` and a global `--transform` expression for stable
  machine-readable output.

Example of a data-plane fallback without a dedicated product command:

```bash
arkcli api arkruntime.create_embeddings \
  --params '{"model":"<embedding-model-id>","input":"BytePlus Ark"}' \
  --transform 'data.0.embedding'
```

Replace the model placeholder with an ID available to the active BytePlus
profile.

## Authentication and verification

- Run `arkcli auth status --format json` before a remote control-plane action.
- Data-plane actions require an active API key resolved through the selected
  BytePlus profile.
- Do not call account-opening or payment-verification actions directly. Use
  the combined `byteplus_sso.identity.verified` contract exposed by
  `arkcli auth status`.
- A billable write must follow the shared BytePlus verification gate and
  explicit confirmation rules.

## Write guard

Raw API invoke mode supports a command-local Client Preview:

```bash
arkcli api <action> --params '<json-object>' --dry-run --format json
```

Inspect `mode=client_preview`, `steps[0].target`, and
`steps[0].payload`. Preview success does not prove authentication,
permission, quota, account verification, Region support, or server-side
validation.

A `"DryRun":true` member inside `--params` is a backend request field, not the
CLI Client Preview flag. Without CLI `--dry-run`, the command still creates a
transport and sends a real network request.

For create, update, delete, activation, credential rotation, or billing
actions:

1. Prefer the guarded product command when one exists.
2. Resolve the exact action and payload.
3. Run the exact invocation once with CLI `--dry-run` and inspect the plan.
4. Show the target and impact to the user.
5. Obtain explicit confirmation.
6. Remove CLI `--dry-run` and invoke only the confirmed payload.

## Pagination and output

- `--page-all` has special Raw API behavior only for `apikey.list`.
- Other actions are passed through once; `--page-all` does not invent or
  mutate their pagination fields.
- Use `--debug` only for diagnosis because it writes request and response
  details to stderr.
- Do not expose tokens or full keys in debug output or copied results.

## Common errors

| Error | Meaning | Recovery |
|---|---|---|
| `unknown action "..."` | The exact name is not registered | Run `arkcli api --list`; do not guess a replacement |
| `invalid --params JSON: ...` | The payload is not a valid JSON object | Validate quoting, commas, and field casing |
| Authentication or permission error | The action reached a remote boundary without usable credentials or permission | Return to `arkcli-auth`; do not retry blindly |
| Unexpected response fields | The assumed contract was wrong or changed | Re-check the exact BytePlus contract before parsing |

## Guard Checklist

- Keep Raw API Explorer as the last fallback.
- Require an exact registered action name and exact BytePlus payload contract.
- Treat registry presence and product support as different facts.
- Route authentication and configuration failures to their owning Skills.
- Use command-local `--dry-run` to preview a raw action, but never treat it as
  server validation or permission verification.
- Never treat payload `"DryRun":true` as a local preview.
- Confirm every write target and impact.
- Use `auth status`, not direct verification actions, for the combined
  BytePlus gate.
- Preserve structured backend errors and request IDs.

## References

- [`references/arkcli-api.md`](references/arkcli-api.md) - detailed command,
  parameter, pagination, authentication, and safety contract
- [`references/evals.md`](references/evals.md) - trigger, fallback, and write
  safety evaluations
- [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md) - authentication and
  verification recovery
- [`../arkcli-config/SKILL.md`](../arkcli-config/SKILL.md) - effective
  profile, Region, project, and base URL diagnosis
