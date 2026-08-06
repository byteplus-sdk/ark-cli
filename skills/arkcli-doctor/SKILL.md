---
name: arkcli-doctor
version: 1.0.0
description: "BytePlus arkcli doctor gateway for CLI, account, error, model, Endpoint and ModelArk/VMP diagnosis, plus ModelArk generation-origin verification for 1-20 media URLs. Origin verification uses doctor +verify-origin: disclose one aggregate batch price, wait for one confirmation covering the whole batch, then relay the complete terminal JSON without interpreting IsOfficial. It does not determine truthfulness, copyright, ownership, legal certification, content safety or media quality. BytePlus has no doctor report."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli doctor --help"
---

# arkcli doctor for BytePlus

Before using this Skill, read
[`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) for product isolation,
structured output, confirmation handling, and safety rules. Doctor has several
diagnostic-first paths that intentionally do not use the shared business-command
authentication gate: local health, account, error and catalog checks execute
directly; model and Endpoint diagnosis also execute directly so their own
authentication, not-found, and VMP findings remain authoritative. Only a real
remote metrics query uses the shared authentication gate.

## Product contract

- Use only the BytePlus build and BytePlus state under `~/.arkcli-bp/`.
- The BytePlus control-plane Region is fixed to `ap-southeast-1`. Never retry a
  failed diagnosis with another product's or another Region's profile.
- BytePlus supports the `en_us` UI locale only.
- `arkcli doctor report` is not registered on BytePlus. Never invoke, preview,
  suggest, or emulate it, and never expect `report_suggestion` in model or
  Endpoint output.
- Every BytePlus doctor JSON object is a subset of the corresponding Volc
  doctor object. Never expect, document, or invent a BytePlus-only output field.
- The default, account, error, model, Endpoint, and metrics paths are read-only.
  The only optional mutation in this Skill is `--auto-bind`, which can reuse or
  create a VMP workspace and bind ModelArk telemetry after explicit approval.
- `--auto-bind` cannot enable VMP, accept service terms, create the
  service-linked role, repair IAM, activate a model, or pay a bill.
- `doctor +verify-origin` is an explicit workflow exception: it creates remote
  asynchronous verification tasks and can be billable. One confirmation covers
  every URL in the current batch.

## When To Trigger

- BytePlus arkcli installation, version, DNS, TLS, clock, profile, or credential
  health needs diagnosis.
- BytePlus real-name verification, balance, IAM, VMP, or TOS
  readiness is unclear.
- A ModelArk or BytePlus OpenAPI error code needs a structured explanation.
- One inference Endpoint or foundation model is unavailable, unhealthy, slow,
  returning errors, or under quota pressure.
- A named ModelArk metric, one metric value, time series, or explicit `ark_*`
  PromQL query is needed from BytePlus VMP.
- The user supplies 1-20 image or video URLs and asks whether they contain
  ModelArk, Seedance, or Seedream generation-origin features.

## When NOT To Trigger

- Installation, login, logout, or credential changes without a diagnosis
  request: route to `arkcli-auth`.
- Profile creation or switching: route to `arkcli-profile` or `arkcli-config`.
- Creating an Endpoint: route to `arkcli-deploy`.
- Starting, stopping, updating, or deleting an existing Endpoint: route to
  `arkcli-infer-endpoint`.
- Pure usage reporting without a root-cause question: route to `arkcli-usage`.
- Foundation-model metadata, parameters, or pricing without a health question:
  route to `arkcli-models` or `arkcli-pricing`.
- Raw OpenAPI exploration: use `arkcli-api-explorer` only after confirming that
  no doctor product command covers the task.
- Any request for `doctor report`: explain that BytePlus does not expose this
  command. Do not substitute another product's workflow or URL.
- Requests to judge whether media content is true, misleading, copyrighted,
  legally owned, safe, compliant, high quality, or playable.

## Command family

```bash
arkcli doctor
arkcli doctor account
arkcli doctor error <code>
arkcli doctor infer-endpoint <endpoint-id> [--window 24h] [--auto-bind]
arkcli doctor model <model-name> [--window 24h] [--auto-bind]
arkcli doctor +verify-origin <media-url> [media-url...]

arkcli doctor metrics list
arkcli doctor metrics describe <query-id>
arkcli doctor metrics <query-id> [--param key=value] [--filter label=value] \
  [--window 1h] [--step 30s] [--workspace-id <uuid>] \
  [--format json|scalar|table] [--auto-bind]
arkcli doctor metrics raw --promql '<ark_* expression>' \
  [--window 1h] [--step 30s] [--workspace-id <uuid>] \
  [--format json|scalar|table] [--auto-bind]
```

Use `--format json` for agent consumption. `metrics list` and
`metrics describe` always return JSON. The local `metrics --format` flag selects
the result shape for a named or raw query; it does not change catalog output.

`--window` uses Go duration syntax. Use `168h`, not `7d`. All executable
Doctor diagnostics are read-only and do not register `--dry-run`; execute the
requested diagnostic directly. `--auto-bind` remains a separately confirmed
mutation when only telemetry workspace binding is missing.

`doctor +verify-origin` is different: it supports a local `--dry-run` workflow
preview, and a new batch requires one aggregate price confirmation before real
execution.

## Origin-verification workflow

For media generation-origin verification, first read
[`references/verify-origin.md`](references/verify-origin.md).

1. Collect all URLs and reject more than 20.
2. Run one CLI command for the entire batch without `--yes`.
3. Show the aggregate USD price disclosure completely.
4. Wait for explicit confirmation after disclosure.
5. Rerun the same batch once with one `--yes`; that confirmation covers every
   URL in the batch.
6. Do not spawn one command per URL, write a shell loop, call raw Actions, or
   implement polling in the Agent.
7. The CLI owns Create/Get pacing and five-second polling.
8. Make the complete terminal stdout JSON the entire final response. Add no
   summary, translation, interpretation, Markdown fence, prefix, or suffix.

Do not interpret `IsOfficial=True`, `False`, or `Null`. This API exposes
generation-origin technical features, not legal certification or a content
truthfulness decision.

## Core workflow: from user message to command

1. Extract the requested action and every identifier before choosing a command:
   `endpoint_id`, foundation `model_name`, `error_code`, `request_id`, metric
   `query_id`, time window, and whether the user asked for a write.
2. Apply the resource-first decision table below. Any Endpoint ID or foundation
   model name routes to its Doctor scope before standalone metrics or error
   lookup because the resource diagnosis preserves status, usage, error
   distribution, and quota evidence.
3. Load only the selected scope reference. Run `doctor model` and
   `doctor infer-endpoint` directly without an authentication or existence
   preflight; those commands preserve the resource context and diagnose their
   own authentication, not-found, and VMP failures.
4. Execute one product command with structured output. Do not add existence
   preflights that duplicate doctor checks.
5. Read stable fields by name, then return the highest-severity finding, its
   evidence and time window, and one safe next step.
6. If a write is required, stop at the confirmation boundary. Only
   `--auto-bind` is in scope here.

## Intent precedence

1. Any Endpoint ID routes to `doctor infer-endpoint`, including when an error
   code or metric request is also present.
2. Any foundation-model name routes to `doctor model`, including when an error
   code or metric request is also present.
3. Without a resource identifier, `metrics list` or `metrics describe` intent
   is local and exact. Run it once without authentication, profile
   initialization, or VMP access, then stop.
4. Without a resource identifier, a named metric value or time series, a known
   query ID, or an explicit `ark_*` PromQL expression routes to
   `doctor metrics`.
5. An error code or error JSON with no resource identifier routes to
   `doctor error`, including authentication- or signature-looking codes.
6. Real-name verification, balance, IAM, VMP, or TOS readiness routes to
   `doctor account`.
7. Installation, DNS, TCP/TLS, clock, configuration, or credential-health
   intent routes to the default `doctor` scope.
8. A `request_id` alone is not reversible through doctor. Preserve it for
   support and ask for the error code, Endpoint ID, or model name.

## Path decision table

Use this table after extracting every identifier. A resource row always wins
over standalone metrics or error intent.

| Available context | Error code present | No error code |
|---|---|---|
| Endpoint ID | Path 1: run `doctor infer-endpoint <id>`, then interpret the matching family in `error-codes.md` | Path 3: run `doctor infer-endpoint <id>` for state, usage, errors, and quota |
| Foundation-model name | Path 1: run `doctor model <name>`, then interpret the matching family in `error-codes.md` | Path 3: run `doctor model <name>` for existence, Endpoint distribution, usage, errors, and account quota |
| No resource ID | Path 2: run `doctor error <code>` and load `error-codes.md` | Route by explicit CLI, account, or metrics intent; otherwise ask for the resource or failure details |
| Request ID only | Preserve the RequestId, ask for the resource and exact code, and do not claim a reverse lookup | Preserve the RequestId and ask for the resource or error response |

Path 1 must not be downgraded to Path 2. For example:

- `ep-example returns RequestBurstTooFast` means run
  `arkcli doctor infer-endpoint ep-example --format json`, not only
  `arkcli doctor error RequestBurstTooFast`.
- `model seed-1-6 returns ModelNotOpen` means run
  `arkcli doctor model seed-1-6 --format json`, not only
  `arkcli doctor error ModelNotOpen`.
- If the user supplies only `MissingSignature`, run
  `arkcli doctor error MissingSignature --format json`.

Do not use `ModelAccessDenied` as a BytePlus example; it is not a registered
BytePlus doctor code. Use the exact returned code, such as `ModelNotOpen`,
`AccessDenied`, or `OperationDenied.PermissionDenied`.

## Scope router

Read only the reference selected by this table before execution.

| User goal | Command | Required reference |
|---|---|---|
| Installation, DNS, TLS, clock, profile, or credentials | `arkcli doctor` | [`references/scope-cli.md`](references/scope-cli.md) |
| Real-name verification, balance, IAM, VMP, or TOS | `arkcli doctor account` | [`references/scope-account.md`](references/scope-account.md) |
| Error meaning without a resource identifier | `arkcli doctor error <code>` | [`references/error-codes.md`](references/error-codes.md) |
| One inference Endpoint | `arkcli doctor infer-endpoint <endpoint-id>` | [`references/scope-infer-endpoint.md`](references/scope-infer-endpoint.md) |
| One foundation model across visible Endpoints | `arkcli doctor model <model-name>` | [`references/scope-model.md`](references/scope-model.md) |
| Catalog discovery, one metric, or PromQL | `arkcli doctor metrics ...` | [`references/scope-metrics.md`](references/scope-metrics.md) |

## Authentication and preflight rules

- Run the default `doctor` scope, `doctor account`, `doctor error`,
  `doctor metrics list`, and `doctor metrics describe` directly. These paths are
  intentionally useful while signed out or are fully local.
- Treat `doctor error` as a complete local lookup. Return its structured result
  or explicit unknown-code failure and stop; do not replace it with auth or
  profile recovery unless the user separately asks to repair credentials.
- Run `doctor model` and `doctor infer-endpoint` directly. Do not run
  `arkcli auth status`, start login, or inspect a profile first. If the Doctor
  result reports an authentication problem, handle that returned result with
  `arkcli-auth` without replacing the original resource diagnosis.
- `metrics list` and `metrics describe` are fully local and need no
  authentication or VMP preflight. Named and raw metrics queries are remote, so
  apply the shared authentication gate before executing them.
- Do not run `arkcli models list`, `arkcli infer endpoint list`, or another
  existence probe before `doctor model` or `doctor infer-endpoint`. Their
  `exists` checks are authoritative for this workflow.
- Doctor read-only scopes do not expose Client Preview. Do not invent or pass
  `--dry-run`; run the requested diagnostic and interpret its structured output.

## Output schema overview

| Command | Stable fields to inspect first |
|---|---|
| `doctor` | `installation`, `connectivity.dns`, `connectivity.tcp`, `connectivity.clock_skew`, `auth`, `configuration` |
| `doctor account` | `identity`, `compliance.realname`, `compliance.balance`, `permissions.iam_system_policies`, `ecosystem.vmp`, `ecosystem.tos` |
| `doctor error` | `code`, `category`, `subtype`, `root_cause`, `hint`, `rules`, `needs_backend`, `skill`, `reference` |
| `doctor infer-endpoint` | `endpoint_id`, `window`, `exists`, `status`, `model_info`, `usage`, `errors`, `error_rate`, `quota_pressure` |
| `doctor model` | `model_name`, `window`, `exists`, `endpoint_total`, `endpoint_state_distribution`, `usage`, `top_endpoint_by_usage`, `errors`, `error_rate`, `top_endpoint_by_error_rate`, `account_quota_pressure` |
| `doctor metrics list` | Array entries containing `id`, `summary`, `metric_family`, `calc_kind`, and `unit` |
| `doctor metrics describe` | Catalog metadata including `template`, declared `params`, and output aggregation |
| Named or raw metrics query | `query_id` when named, rendered `promql`, `window`, and exactly one of `series`, top-level scalar fields, or `table` |

Treat `ok`, `available`, `not_found`, `reason`, `type`, `level`, and nullable
fields as authoritative. Do not infer success from process exit alone and do
not turn an unavailable or omitted value into zero. After a successful VMP
query, an empty series or zero can mean no matching traffic in that workspace,
label set, and window; it is not a health verdict by itself.

`exists.ok` proves only that the model is visible in the foundation-model
catalog. Doctor does not expose a BytePlus-only activation object or infer
activation and overdue status from catalog visibility.

## Error and pressure interpretation

Use the subtype returned by `doctor error` and the sections in
[`references/error-codes.md`](references/error-codes.md):

| Returned family or subtype | Reference section |
|---|---|
| BytePlus gateway signature, authorization, validation, or infrastructure | Official BytePlus common OpenAPI errors |
| `model_access_denied` | ModelArk access errors |
| `rate_limit_exceeded` | Rate limits and burst protection |
| `account_status` | Account and billing errors |
| `validation_error` or `not_found` | Validation and resource errors |
| Content-safety and policy subtypes | Content safety |

Error-rate output uses `threshold_percent=5`. Quota pressure uses the command's
structured `level`:

| Percent | Level | Meaning |
|---:|---|---|
| `>=95` | `critical95` | At or near the limit; reduce traffic or request more quota |
| `>=80` | `warn80` | High pressure |
| `>=50` | `warn50` | Moderate pressure |
| `<50` | `pass` | Below the configured warning thresholds |

Do not retry authentication, permission, validation, or payment errors blindly.
Retry transient gateway or overload errors only when the operation is
idempotent, with exponential backoff, while preserving `RequestId`.

## VMP dependency and `--auto-bind`

Endpoint, model, named metric, and raw PromQL paths use BytePlus VMP. Without an
explicit `--workspace-id`, a successful remote path requires:

1. VMP enabled for the account.
2. ModelArk cross-service authorization through the service-linked role.
3. A VMP workspace bound to ModelArk telemetry.

If only step 3 is missing, show the exact command with `--auto-bind`, explain
that it may reuse the first workspace or create `ark_default` and bind
telemetry, and obtain explicit approval before execution. If step 1 or 2 is
missing, stop at the official BytePlus setup path; adding `--auto-bind` cannot
repair it.

An explicit `--workspace-id` queries that workspace directly and skips the
ModelArk telemetry precheck. Do not invent or copy a workspace ID from another
account.

Official BytePlus references:

- VMP overview: https://docs.byteplus.com/en/docs/vmp/Monitoring-overview
- VMP quick start: https://docs.byteplus.com/en/docs/vmp/Quick-start
- ModelArk access control: https://docs.byteplus.com/en/docs/ModelArk/1263493

## Guard Checklist

- Confirm the executable and installed Skill are from the BytePlus product.
- Keep the active BytePlus account, profile, fixed Region, and environment.
- Use JSON fields rather than parsing tables or localized prose.
- Do not expose credentials, signed headers, or full keys.
- Preserve exact error codes and `RequestId` values.
- Do not invent unregistered flags such as `--fix`.
- Do not run model or Endpoint existence preflights outside doctor.
- Do not run an authentication preflight before `doctor model` or
  `doctor infer-endpoint`.
- Obtain explicit approval before adding `--auto-bind`.
- Never route to or emulate `doctor report` on BytePlus.
- For origin verification, obtain exactly one confirmation for the whole batch,
  never one confirmation per media URL.

## Output handoff

Return, in order:

1. The highest-severity finding or the explicit unknown state.
2. The exact structured evidence fields and time window.
3. The BytePlus-specific cause, without substituting another product's
   semantics.
4. One safe next command or official BytePlus page.
5. The preserved `RequestId` when upstream returned one.

Do not repeat the full JSON unless the user asks. Never replace an unknown or
unavailable result with a successful result.

Exception: for `doctor +verify-origin`, always relay the complete stdout JSON
as the entire final response because the upstream legal wording and every
response field must remain intact.

## References

- [`references/scope-cli.md`](references/scope-cli.md)
- [`references/scope-account.md`](references/scope-account.md)
- [`references/error-codes.md`](references/error-codes.md)
- [`references/scope-infer-endpoint.md`](references/scope-infer-endpoint.md)
- [`references/scope-model.md`](references/scope-model.md)
- [`references/scope-metrics.md`](references/scope-metrics.md)
- [`references/verify-origin.md`](references/verify-origin.md)
- [`references/evals.md`](references/evals.md)
- [`CONTRIBUTING.md`](CONTRIBUTING.md)
- [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md)
- [`../arkcli-config/SKILL.md`](../arkcli-config/SKILL.md)
- [`../arkcli-deploy/SKILL.md`](../arkcli-deploy/SKILL.md)
- [`../arkcli-infer-endpoint/SKILL.md`](../arkcli-infer-endpoint/SKILL.md)
- [`../arkcli-usage/SKILL.md`](../arkcli-usage/SKILL.md)
