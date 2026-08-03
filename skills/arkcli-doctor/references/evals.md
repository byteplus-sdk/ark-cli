# Evaluation prompts

These documentation evals lock BytePlus doctor routing, command contracts,
structured-output interpretation, and product boundaries. They are not a
substitute for BytePlus-tagged command tests.

## Trigger coverage

### Local metric catalog

Prompt:

`List the named BytePlus doctor metrics without accessing VMP.`

Expected:

- Runs `arkcli doctor metrics list --format json` exactly once.
- Does not run an authentication preflight or require a configured profile.
- Does not inspect help, configuration, or VMP state afterward.
- Does not hardcode the catalog count.

### Local metric description

Prompt:

`Show the parameters for BytePlus doctor metric request.error.rate. Do not query VMP.`

Expected:

- Runs `arkcli doctor metrics describe request.error.rate --format json`.
- Reads the declared parameter name `endpoint`, not `endpoint_id`.
- Stops after the local catalog result.

### CLI health while signed out

Prompt:

`Check whether this BytePlus arkcli installation can reach its configured service. Do not log in or call business APIs.`

Expected:

- Runs `arkcli doctor --format json` without an auth preflight.
- Does not inspect or mutate profiles and does not initialize configuration.
- Reads installation, DNS, TCP/TLS, clock, auth, and configuration fields.
- Does not treat a reachable host plus later HTTP 401/403 as DNS or TLS failure.
- Keeps the BytePlus profile and fixed Region, and never probes another
  product's host when no profile is configured.

### Real-name verification and account readiness

Prompt:

`Check whether my BytePlus account has passed real-name verification and inspect account readiness.`

Expected:

- Runs `arkcli doctor account --format json`.
- Reads `compliance.realname.verified` as the `GetVerifyInfo.IsVerified`
  real-name-verification result.
- Does not infer real-name verification from account-opening or
  payment-qualification statuses.
- Reads available balance and currency, IAM evidence, VMP, and TOS.
- Does not expect raw readiness enums, extended balance fields,
  `ecosystem.model_ark`, activation, or billing-overview data.

### Public gateway error code

Prompt:

`I only have BytePlus error code MissingSignature. Explain it with arkcli.`

Expected:

- Runs `arkcli doctor error MissingSignature --format json` without auth.
- Explains the missing BytePlus OpenAPI signature.
- Preserves the exact code and any RequestId supplied by the user.
- Uses only BytePlus documentation or console links.

### Authentication-looking unknown error JSON

Prompt:

`Here is a BytePlus error JSON with InvalidApiKey.NotFound. Use arkcli to diagnose it.`

Expected:

- Runs `arkcli doctor error InvalidApiKey.NotFound --format json` directly.
- Preserves the CLI's explicit unknown-code result; it does not claim a
  successful lookup or rewrite the code to a similar registered code.
- Does not run auth status, login, API-key commands, profile inspection, or
  configuration initialization as a substitute for the requested diagnosis.

### Endpoint resource-first routing

Prompt:

`Endpoint ep-example returns RequestBurstTooFast. Diagnose it.`

Expected:

- Runs `arkcli doctor infer-endpoint ep-example --format json` directly before
  any standalone error lookup.
- Does not run `arkcli auth status`, start login, inspect a profile, or run an
  Endpoint/model list preflight.
- Uses the Endpoint window, state, modality, usage availability, VMP error
  channels, and quota evidence.
- Loads the rate-limit section of `error-codes.md` for the exact code.

### Model resource-first routing

Prompt:

`BytePlus model seed-1-6 returns ModelNotOpen. Diagnose the model.`

Expected:

- Runs `arkcli doctor model seed-1-6 --format json` directly before a
  standalone lookup.
- Does not run `arkcli auth status`, start login, inspect a profile, or run a
  model/Endpoint list preflight.
- Reads `exists`, visible Endpoint distribution, usage, errors, top Endpoints,
  and account quota.
- Does not expect or invent a BytePlus-only `activation` field.
- Loads the ModelArk access section for `ModelNotOpen`.

### Exact model activation without an error code

Prompt:

`Check whether BytePlus model seed-1-6 is activated for this account and whether payment is overdue.`

Expected:

- Runs `arkcli doctor model seed-1-6 --format json`.
- States that the Doctor schema does not expose exact activation or overdue
  state.
- Uses `exists` and Endpoint evidence only for what those fields prove.
- Does not call `GetModelChargeItem` or infer activation from model existence
  or Endpoint count.

### Named metric query

Prompt:

`Get the aggregate BytePlus request.error.rate over the last hour as one scalar value.`

Expected:

- Uses the catalog entry `request.error.rate` with its default all-resource
  selectors.
- Runs a command equivalent to:

```bash
arkcli doctor metrics request.error.rate \
  --window 1h \
  --format scalar
```

- Reads top-level `value`, `unit`, and `aggregation`.
- Does not call raw PromQL when the named query covers the task.

### Resource ID outranks metric intent

Prompt:

`Get the BytePlus error rate for Endpoint ep-example over the last hour.`

Expected:

- Runs `arkcli doctor infer-endpoint ep-example --window 1h --format json`
  before any standalone metrics query.
- Uses the Endpoint diagnosis error-rate channel and preserves its status,
  usage, error distribution, and quota evidence.
- Does not downgrade the resource request to only
  `doctor metrics request.error.rate`.

### Unsupported metric preview

Prompt:

`Preview the aggregate BytePlus request.error.rate over the last hour. Do not access VMP.`

Expected:

- States that Doctor metrics queries do not expose `--dry-run`.
- Does not invent the flag and does not access VMP.
- May use the local `doctor metrics describe request.error.rate --format json`
  command to explain the catalog request shape without claiming a Client
  Preview.

### RequestId-only path

Prompt:

`BytePlus request req-123 failed. Find the root cause, but I have no code, model, or Endpoint ID.`

Expected:

- States that doctor has no RequestId reverse lookup.
- Preserves `req-123` and asks for the error response or resource identifier.
- Does not guess a model, Endpoint, error code, account, or metric path.

## Anti-trigger coverage

### Report boundary

Prompt:

`A BytePlus generated video has misspelled subtitles. Run doctor report.`

Expected:

- States that `arkcli doctor report` is unavailable on BytePlus.
- Does not invoke, preview, suggest, or emulate that command.
- Does not use another product's model names, reporting URLs, or semantics.
- Offers error, model, or Endpoint diagnosis only when the required identifier
  is available or asks for it.

### Deployment boundary

Prompt:

`Create a new BytePlus inference Endpoint for my model.`

Expected:

- Does not use doctor as the destination Skill.
- Routes to `arkcli-deploy` and preserves the original deployment goal.

### Endpoint lifecycle boundary

Prompt:

`Restart BytePlus Endpoint ep-example.`

Expected:

- Does not use doctor unless the user also asks for diagnosis.
- Routes the lifecycle operation to `arkcli-infer-endpoint`.

### Pure usage boundary

Prompt:

`Show my BytePlus token usage for this month. I do not need root-cause analysis.`

Expected:

- Routes to `arkcli-usage`.
- Does not substitute model or Endpoint doctor output.

### Profile mutation boundary

Prompt:

`Switch this BytePlus session to another profile.`

Expected:

- Routes to `arkcli-profile` or `arkcli-config`.
- Does not treat the profile change as a doctor remediation.

### Unknown error-code boundary

Prompt:

`Explain BytePlus error code ModelAccessDenied.`

Expected:

- Does not claim that the unregistered code is a successful lookup.
- Explains that the exact BytePlus response is required and gives valid
  examples such as `ModelNotOpen` or `AccessDenied` without rewriting the
  user's evidence.

## Guard behavior

### VMP workspace mutation

Prompt:

`My BytePlus model diagnosis says vmp_workspace_unbound. Fix it.`

Expected:

- Explains that `--auto-bind` may reuse or create a workspace and bind ModelArk
  telemetry.
- Shows the exact model command and requests explicit approval before running.
- Does not claim that `--auto-bind` enables VMP or grants the service-linked
  role.

### VMP not enabled

Prompt:

`doctor model says vmp_not_open. Add --auto-bind and keep retrying.`

Expected:

- Refuses the invalid retry loop.
- Explains that `--auto-bind` cannot enable VMP or accept terms.
- Sends the user to the official BytePlus VMP setup path.

### Explicit workspace

Prompt:

`Query request.error.rate in BytePlus VMP workspace ws-user-supplied.`

Expected:

- Uses the user-supplied `--workspace-id ws-user-supplied`.
- Explains that the explicit workspace bypasses telemetry workspace resolution.
- Does not copy, discover, or substitute a workspace from another account.

### Structured unknown state

Prompt:

`The BytePlus model exists and has no visible Endpoint. Is the model definitely not activated?`

Expected:

- Answers that activation is unknown from those fields.
- Explains that catalog visibility and Endpoint count do not prove activation.
- Does not invent an `activation` object or an overdue result.

## Happy-path commands

Local lookup:

```bash
arkcli doctor error MissingSignature --format json
```

Expected:

- Returns a structured BytePlus common-auth diagnosis.
- Preserves the exact error code.

Remote model diagnosis:

```bash
arkcli doctor model seed-1-6 --format json
```

Expected:

- Executes the read-only diagnosis directly without inventing `--dry-run`.
- Does not include `GetModelChargeItem` or any BytePlus-only output field.
- Does not advertise `doctor report`.
