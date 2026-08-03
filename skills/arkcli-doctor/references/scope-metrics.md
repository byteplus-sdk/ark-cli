# BytePlus VMP metrics scope

Use this reference when the user needs the live metric catalog, one ModelArk
metric value, a time series, or explicit PromQL. Use model or Endpoint diagnosis
for an overall health decision; metrics output does not assign health by itself.

Follow the resource-first rule in the parent Skill before entering this scope.
If the request contains an Endpoint ID or foundation-model name, run
`doctor infer-endpoint` or `doctor model` first even when the user also names a
metric. Use this metrics scope directly only when no resource identifier is
present, or after the resource diagnosis when a separate raw metric value is
still required.

## Command families

| Goal | Command | Remote access |
|---|---|---|
| List the live named-query catalog | `arkcli doctor metrics list --format json` | No; local embedded catalog only |
| Describe one query and its parameters | `arkcli doctor metrics describe <query-id> --format json` | No; local embedded catalog only |
| Run a named query | `arkcli doctor metrics <query-id> ... --format json|scalar|table` | Yes; BytePlus VMP |
| Run raw PromQL | `arkcli doctor metrics raw --promql '<expr>' --format json|scalar|table` | Yes; BytePlus VMP |

## Discover the live catalog

Do not hardcode a query count or parameter list. The embedded BytePlus catalog
can change by release.

```bash
arkcli doctor metrics list --format json
arkcli doctor metrics describe request.error.rate --format json
```

These commands do not require login, a configured profile, VMP, or an auth
preflight. Execute the requested catalog command once, return its result, and
stop. Do not inspect configuration, command help, or VMP state after a
successful local catalog read.

`list` returns an array of:

- `id`
- `summary`
- `metric_family`
- `calc_kind`
- `unit`

`describe` additionally returns the PromQL `template`, declared `params` with
types and defaults, output `unit`, and `aggregation_hint`. Use the exact
parameter names from `describe`; unknown names are rejected.

## Run a named query

Apply the shared authentication gate, describe the query when its parameters
are not already known, and then run it:

```bash
arkcli doctor metrics request.error.rate \
  --param endpoint=ep-example \
  --window 1h \
  --format json
```

Do not use `--param endpoint_id=...` for this query; the catalog declares the
parameter as `endpoint`.

Useful flags:

| Flag | Contract |
|---|---|
| `--param key=value` | Bind one declared catalog parameter; repeatable |
| `--filter label=value` | Add a label matcher only when the template declares `${@filters}`; repeatable |
| `--window <duration>` | Lookback window in Go duration syntax; default `1h` |
| `--step <duration>` | Sample step; default is window/60 clamped to 10-60 seconds |
| `--workspace-id <uuid>` | Query a specific BytePlus VMP workspace and skip telemetry workspace resolution |
| `--format json` | Return rendered PromQL, window, and raw series |
| `--format scalar` | Reduce the matrix using the catalog aggregation hint and return top-level `value`, `unit`, and `aggregation` |
| `--format table` | Return one row per series with tags, latest, mean, and max |
| `--auto-bind` | Reuse/create a workspace and bind ModelArk telemetry when only binding is missing; requires approval |

Doctor metrics queries are read-only and do not register `--dry-run`. The local
`metrics list` and `metrics describe` commands expose the catalog and request
shape without VMP access. A named or raw query is the real remote diagnosis and
must not be presented as a Client Preview.

## Output interpretation

Named and raw results share these fields:

| Field | Meaning |
|---|---|
| `query_id` | Present for a named query; omitted for raw PromQL |
| `promql` | Exact rendered or supplied query sent to VMP |
| `window.start`, `window.end`, `window.step` | Effective UTC query bounds |
| `series` | JSON matrix output |
| `value`, `unit`, `aggregation` | Scalar output at the top level |
| `table` | Per-series summary rows for table output |

Preserve the rendered PromQL, workspace scope, label values, and time window
when reporting a query issue. An empty `series`, empty `table`, or zero scalar
after a successful query means no matching data for that selection or a
legitimate zero; it is not proof that ModelArk is healthy or unhealthy.

Before changing the query, verify:

1. The query ID and declared parameter names.
2. The Endpoint/model label values.
3. The BytePlus VMP workspace and ModelArk telemetry binding.
4. The selected window and expected traffic.
5. The metric family for the workload modality.

## Raw PromQL

Use raw PromQL only when the live catalog cannot express the requested metric:

```bash
arkcli doctor metrics raw \
  --promql 'sum(rate(ark_api_proxy_request_total[5m]))' \
  --window 1h \
  --format json
```

The expression must mention at least one `ark_*` metric. General cluster
queries such as `node_*`, `kube_*`, or `up{}` are rejected. Raw mode bypasses
catalog parameter validation and has no catalog unit; verify the expression
carefully and do not use it as the default path.

## VMP workspace and `--auto-bind`

Without `--workspace-id`, arkcli resolves the workspace from ModelArk telemetry
configuration and checks VMP subscription plus cross-service authorization.

If only workspace binding is missing, explain the mutation and obtain approval:

```bash
arkcli doctor metrics request.error.rate \
  --param endpoint=ep-example \
  --auto-bind \
  --format json
```

If VMP is disabled or the service-linked role is missing, do not add
`--auto-bind`. Use the official BytePlus setup path:

- https://docs.byteplus.com/en/docs/vmp/Quick-start
- https://docs.byteplus.com/en/docs/ModelArk/1263493

An explicit `--workspace-id` bypasses ModelArk telemetry workspace resolution.
Use only a workspace the user supplied or that belongs to the active BytePlus
account; never copy one from another product or account.

## Boundaries

- Catalog list and describe are local and read-only.
- Named and raw queries are remote but read-only unless `--auto-bind` is added.
- Metrics output is evidence, not a complete model or Endpoint health decision.
- Do not hardcode the current catalog size.
- Do not use raw PromQL when a named catalog query exists.
- Do not query non-`ark_*` infrastructure metrics through this command.
