# Raw API Explorer reference

Read [`../SKILL.md`](../SKILL.md) first. Raw API Explorer is a low-level
fallback for registered actions; it does not replace stable BytePlus product
commands.

## Command surface

```bash
# List registered action descriptors
arkcli api --list
arkcli api

# Invoke one registered action
arkcli api <registered-action> --params '<json-object>'

# Preview one invocation locally without contacting BytePlus
arkcli api <registered-action> --params '<json-object>' --dry-run
```

| Input | Required | Contract |
|---|---|---|
| `<registered-action>` | No in list mode; yes in invoke mode | Exact name returned by `arkcli api --list` |
| `--list` | No | Lists descriptors and performs no remote request |
| `--params` | No | JSON object string; default `{}` |
| `--dry-run` | No | Invoke mode only; emits a local `preview.v1` plan and sends no request |
| `--format` | No | Global output format |
| `--transform` | No | Global GJSON-style result extraction |
| `--debug` | No | Sends diagnostic request and response details to stderr |

List output contains:

- `name`: registered action name;
- `protocol`: client transport family;
- `target`: underlying operation target.

These fields are client descriptors, not proof of BytePlus account or service
support.

Client Preview output includes `mode=client_preview` and an API step containing
the operation `protocol`, `target`, and redacted `payload`. It does not prove
that the account can execute the action. List mode does not accept
`--dry-run`, because listing the local registry already has no remote effect.

## How to determine `--params`

Use this order:

1. Check whether a product command and capability Skill already own the task.
2. Confirm the exact action name with `arkcli api --list`.
3. Read the matching BytePlus API contract or installed capability reference.
4. If working inside the Ark CLI repository, inspect the registered request
   type under `internal/apis/<domain>/`.
5. Preserve the documented JSON field names, casing, nesting, and value types.
6. Start with the smallest valid payload and add only required fields.

If no authoritative contract is available, stop instead of guessing.

## Data-plane fallback example

Embedding currently has a registered data-plane action and no dedicated
product command:

```bash
arkcli api arkruntime.create_embeddings \
  --params '{"model":"<embedding-model-id>","input":"BytePlus Ark"}'
```

The active BytePlus profile must supply a usable API key. Use a model ID
available to that profile.

For stable downstream processing:

```bash
arkcli api arkruntime.create_embeddings \
  --params '{"model":"<embedding-model-id>","input":"BytePlus Ark"}' \
  --format json \
  --transform 'data.0.embedding'
```

## Authentication contract

- `arkcli api --list` is local and needs no login.
- Control-plane invocation uses the authenticated BytePlus identity.
- Data-plane invocation uses the API key resolved from the active profile.
- A `401` or permission error is not evidence that an action is missing.
  Diagnose with `arkcli auth status --format json`.
- Do not expose full keys, tokens, or secret-bearing payload fields.

## Verification contract

Do not reconstruct the BytePlus account gate by invoking its underlying
actions. The supported public contract is:

```text
arkcli auth status --format json
  -> byteplus_sso.identity.verified
```

Apply the combined gate before a billable write. If the field is `false`, stop
at `https://console.byteplus.com/user/basics/`. If it is absent, treat the
result as unknown and preserve any later structured backend error.

## Pagination contract

Raw API Explorer is pass-through by default.

`apikey.list` is the only action with special global `--page-all` handling:

```bash
arkcli api apikey.list \
  --params '{"PageSize":100}' \
  --page-all \
  --format json
```

- `PageSize` must be a positive integer.
- The CLI increments `PageNumber`, aggregates `Result.Items`, and preserves the
  server `TotalCount`.
- `--page-limit` and `--page-delay` apply to this special case.
- Other actions are invoked once and their payload is not modified by
  `--page-all`.

## Write and deletion risk

Generic Raw API invocation has a command-local Client Preview layer:

```bash
arkcli api <registered-action> \
  --params '<json-object>' \
  --dry-run \
  --format json
```

The CLI does not create a transport in this mode. Review
`steps[0].target` and `steps[0].payload` before a write.

A payload member such as `"DryRun":true` is a server-side request field.
Without the CLI `--dry-run` flag, the raw invocation still sends a network
request.

Before any create, update, delete, activation, credential, or billing action:

1. Prefer a guarded product command.
2. Verify the exact payload and target.
3. Run the exact invocation with CLI `--dry-run`.
4. Explain the target and impact shown by the plan.
5. Obtain explicit user confirmation.
6. Remove CLI `--dry-run` and invoke the confirmed payload exactly once.

Never use Raw API Explorer to bypass a product command's safety gate.

## Error handling

| Symptom | Recovery |
|---|---|
| `unknown action` | Re-list registered actions and use an exact name |
| `invalid --params JSON` | Validate a top-level JSON object and shell quoting |
| Missing required field | Re-read the BytePlus request contract |
| Wrong response path | Inspect structured output before choosing `--transform` |
| `401` or expired identity | Return to `arkcli-auth` |
| Wrong profile, Region, project, or base URL | Return to `arkcli-config` |
| Billable write rejected by verification | Preserve the structured error and use the BytePlus account settings URL |
