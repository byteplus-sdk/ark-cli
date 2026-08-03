# BytePlus inference Endpoint scope

Use this reference whenever the user provides an inference Endpoint ID and
wants health, error, usage, or quota diagnosis. The Endpoint ID outranks a
standalone error-code lookup.

## Commands

```bash
# Default 24-hour diagnosis.
arkcli doctor infer-endpoint <endpoint-id> --format json

# Explicit window.
arkcli doctor infer-endpoint <endpoint-id> --window 6h --format json
```

`--window` uses Go duration syntax such as `30m`, `1h`, `24h`, or `168h`.
`7d` is not accepted.

This diagnostic is read-only and supports neither `--fix` nor `--dry-run`.
Apply remediation through the owning command after explicit confirmation.

Run the Doctor command directly. Do not run `arkcli auth status`, start login,
inspect a profile, call `arkcli infer endpoint list`, call
`arkcli models list`, or add another preflight first. Doctor preserves the
original Endpoint ID and reports its own authentication, not-found, and VMP
failures.

Preset and user-created Endpoint IDs can use different detail actions
internally. Always pass the unchanged ID and let the product command choose.

## Checks and schema

| Output | Meaning |
|---|---|
| `endpoint_id` | Exact requested Endpoint ID |
| `window.duration` | Effective diagnostic window |
| `window.start_time`, `window.end_time` | RFC3339 evidence bounds |
| `exists.ok` | Whether the Endpoint is visible in the active BytePlus profile |
| `exists.not_found` | Normalized missing-resource result |
| `exists.reason` | Lookup or visibility failure |
| `status.healthy` | Whether the lifecycle state is healthy |
| `status.status` | Raw Endpoint lifecycle state |
| `status.reason` | Backend state reason when present |
| `model_info.model_type` | `FoundationModel`, `CustomModel`, or `ServiceModel` |
| `model_info.name`, `model_info.version` | Foundation or service model identity |
| `model_info.custom_model_id` | Custom-model identity when applicable |
| `model_info.modality` | `text`, `image`, `video`, or `unknown` |
| `usage.available` | Whether ModelArk usage data was returned |
| `usage.total_calls` | Calls in the selected window |
| `usage.total_tokens`, `input_tokens`, `output_tokens` | Token values when the modality uses tokens |
| `usage.cache_hit_rate_percent` | Cache-hit ratio for applicable text models |
| `errors.modality` | Metric family selected from authoritative model tags |
| `errors.overall.by_error_code` | Text/image error distribution when applicable |
| `errors.task.by_error_code` | Async task error distribution for video-like workloads |
| `error_rate.threshold_percent` | Current warning threshold, normally 5 |
| `error_rate.overall`, `request`, `task` | Modality-specific error-rate channels |
| `quota_pressure.modality` | Quota family selected for this model |
| `quota_pressure.rpm`, `tpm`, `ipm`, `concurrency` | Applicable limit, peak, percent, and level fields |

The scope does not emit `report_suggestion` on BytePlus.

## Analysis order

1. Confirm `exists.ok`. If false, report `not_found` and `reason`, then stop.
   Check the exact ID, active BytePlus profile, IAM visibility, and fixed Region;
   do not retry in another product.
2. Read `status.healthy`, `status.status`, and `status.reason`. A stopped or
   transitioning Endpoint can explain failures before metrics are considered.
3. Read `model_info.model_type` and `modality`. Do not interpret token, image,
   task, or concurrency fields without this context.
4. Read `usage.available` before all usage numbers. An unavailable probe is not
   zero traffic.
5. Read the error-rate channel for the modality and compare it with
   `threshold_percent`. Then sort the matching `by_error_code` distribution by
   count or percent.
6. Load [`error-codes.md`](error-codes.md) for the leading exact code. Preserve
   any upstream `RequestId`.
7. Read only the quota metrics present for the modality. Use `level` and
   `percent`; do not parse display text.
8. If the problem remains model-wide, run `doctor model <model-name>`. Do not
   broaden the scope before the Endpoint evidence points there.

## Modality routing

| Modality | Error fields | Quota fields |
|---|---|---|
| Text | `errors.overall`, `error_rate.overall` | RPM and TPM when available |
| Image | `errors.overall`, `error_rate.overall` | IPM when available |
| Video or async content generation | `errors.task`, `error_rate.request`, `error_rate.task` | Create-task RPM and concurrency when available |
| Unknown | Preserve the reported fields without guessing a metric family | Preserve present fields and state the limitation |

A peak of zero after a successful VMP query can mean no matching series, no
traffic, or a metric that is not populated for that modality. Do not claim
unused quota without checking availability, window, and surrounding evidence.

## Common finding to action

| Finding | Next action |
|---|---|
| `exists.not_found=true` | Verify the exact Endpoint ID, active BytePlus profile, fixed Region, and IAM visibility |
| `status.healthy=false` | Follow the lifecycle reason; use `arkcli-infer-endpoint` for an authorized start or update operation |
| `usage.available=false` | Report `usage.reason`; do not replace totals with zero |
| Error rate exceeds the threshold | Use the leading exact code and `error-codes.md`; distinguish request and async-task channels |
| `level=critical95` or `warn80` | Reduce traffic, rebalance work, or request the relevant BytePlus quota |
| `RequestBurstTooFast` | Ramp traffic gradually and retry idempotently with exponential backoff |
| `ModelNotOpen` | Run `doctor model <model-name>` for model-wide evidence and interpret the exact code with `error-codes.md` |

## VMP precheck and binding

VMP is required for the monitoring portion. Without an explicit workspace, the
command verifies VMP subscription, the ModelArk service-linked role, and the
telemetry workspace binding.

If and only if the structured error says the workspace is unbound, explain the
write and obtain approval before running:

```bash
arkcli doctor infer-endpoint <endpoint-id> --window 24h --auto-bind --format json
```

The command may reuse the first workspace or create `ark_default`, then bind
ModelArk telemetry. It cannot enable VMP, accept terms, or grant IAM. If VMP is
disabled or the role is missing, stop at the official BytePlus setup path.

## Boundaries

- Read-only unless `--auto-bind` is explicitly approved.
- Does not invoke the Endpoint and does not consume model tokens.
- Does not start, stop, update, or delete the Endpoint.
- Does not compare another account, Region, or product.
- Does not support or suggest `doctor report`.
