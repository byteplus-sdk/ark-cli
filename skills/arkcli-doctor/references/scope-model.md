# BytePlus foundation-model scope

Use this reference when the user provides a BytePlus ModelArk foundation-model
name and wants health, usage, error, Endpoint, or quota diagnosis. The model
name outranks a standalone error-code lookup.

## Commands

```bash
# Default 24-hour diagnosis.
arkcli doctor model <model-name> --format json

# Explicit window.
arkcli doctor model <model-name> --window 6h --format json
```

`--window` uses Go duration syntax. Use `168h`, not `7d`.

This diagnostic is read-only and supports neither `--fix` nor `--dry-run`.
Apply remediation through the owning command after explicit confirmation.

Run the Doctor command directly. Do not run `arkcli auth status`, start login,
inspect a profile, call `arkcli models list`, call
`arkcli infer endpoint list`, or add another preflight first. The model scope
preserves the requested model and reports its own authentication, catalog,
Endpoint-visibility, and VMP failures.

## Output contract

BytePlus uses only fields already present in the Volc `doctor model` schema.
The scope combines foundation-model catalog visibility, usage, Endpoint APIs,
rate-limit APIs, and BytePlus VMP operational evidence. It does not emit a
BytePlus-only `activation` object or `report_suggestion`.

`exists.ok=true` proves only catalog visibility. It does not prove that a model
is activated for the account, that payment is current, or that inference will
succeed.

## Checks and schema

| Output | Meaning |
|---|---|
| `model_name` | Exact requested foundation-model name |
| `window.duration`, `start_time`, `end_time` | Effective diagnostic evidence window |
| `exists.ok`, `not_found`, `reason` | Catalog visibility result |
| `exists.modality` | `text`, `image`, `video`, or `unknown` from authoritative model tags |
| `exists.vendor`, `display_name`, `primary_version` | Returned model metadata |
| `endpoint_total.total` | Visible Endpoints serving the model |
| `endpoint_state_distribution.buckets` | Endpoint lifecycle count and percent by state |
| `usage.available` | Whether model usage was returned |
| `usage.total_calls`, token fields, cache fields | Window totals for the applicable modality |
| `top_endpoint_by_usage` | Top Endpoints from authoritative ModelArk usage data |
| `errors.overall` or `errors.task` | Modality-specific error-code distribution |
| `error_rate.overall`, `request`, or `task` | Modality-specific rate channels |
| `top_endpoint_by_error_rate` | BytePlus VMP ranking using `ark_endpoint` and `base_model` labels |
| `account_quota_pressure` | Applicable account-level limit, peak, percent, and level |

## Analysis order

1. Read `exists.ok`. If false, report `not_found` and `reason`, then stop.
2. Read `exists.modality` before interpreting tokens, images, async tasks, or
   concurrency.
3. Read Endpoint count and state distribution. Zero visible Endpoints means no
   visible deployed Endpoint in the active profile scope; it does not prove an
   activation or payment state.
4. Read `usage.available`, totals, and `top_endpoint_by_usage`. Text models rank
   naturally by tokens; image and video models can fall back to call or task
   count.
5. Read the model-level error-rate channel, leading error codes, and
   `top_endpoint_by_error_rate` for the same window.
6. Load [`error-codes.md`](error-codes.md) for the exact leading code.
7. Read only quota fields present for the modality and use their structured
   `level` and `percent`.
8. Drill into one Endpoint with `doctor infer-endpoint <id>` only when the model
   result identifies a relevant candidate.

## Operational interpretation

| Finding | Next action |
|---|---|
| `exists.ok=false` | Check the exact name, fixed Region, and IAM visibility |
| Many Endpoints are not running | Select the relevant Endpoint and use `doctor infer-endpoint`; lifecycle writes belong to `arkcli-infer-endpoint` |
| One Endpoint dominates usage | Check that Endpoint's state, error rate, and quota pressure |
| One Endpoint dominates error rate | Run `doctor infer-endpoint <id>` for resource-level evidence |
| Model-wide error rate exceeds `threshold_percent` | Interpret the leading exact error codes and distinguish model/account causes from one Endpoint |
| `level=critical95` or `warn80` | Reduce or redistribute traffic, or request the applicable BytePlus quota |
| `ModelNotOpen` | Interpret the exact error with `error-codes.md`; do not invent an activation field |
| `RequestBurstTooFast` | Ramp traffic gradually and retry idempotently with exponential backoff |

## Modality routing

| Modality | Error fields | Usage and quota interpretation |
|---|---|---|
| Text | `errors.overall`, `error_rate.overall` | Tokens, cache, RPM, and TPM when populated |
| Image | `errors.overall`, `error_rate.overall` | Calls, images, and IPM when populated |
| Video or async content generation | `errors.task`, `error_rate.request`, `error_rate.task` | Calls, tasks, create-task RPM, and concurrency when populated |
| Unknown | Preserve returned fields and state the limitation | Do not guess a metric or quota family from the model name |

An empty VMP series or zero peak is not proof of health. Check the workspace,
labels, selected time window, usage availability, and whether that metric is
populated for the modality.

## VMP precheck and binding

If VMP and the ModelArk service-linked role are ready but only the workspace
binding is missing, explain the mutation and obtain approval before running:

```bash
arkcli doctor model <model-name> --window 24h --auto-bind --format json
```

The command may reuse the first workspace or create `ark_default`, then bind
ModelArk telemetry. It does not enable VMP, grant IAM, activate the model, or
make a payment. If VMP is disabled or the service-linked role is missing, stop
at the official BytePlus setup path.

## Boundaries

- Read-only unless `--auto-bind` is explicitly approved.
- Does not invoke the model and does not consume inference tokens.
- Diagnoses foundation models, not custom-model training jobs.
- Does not prove or change model activation or payment state.
- Does not modify Endpoints.
- Does not compare accounts, Regions, or products.
- Does not support or suggest `doctor report`.

Use [`scope-infer-endpoint.md`](scope-infer-endpoint.md) for one Endpoint and
[`scope-account.md`](scope-account.md) for available-balance, IAM, VMP, or TOS
readiness. Doctor does not determine payment qualification, bill status, or
overdue state.
