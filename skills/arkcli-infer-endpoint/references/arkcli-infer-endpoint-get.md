# arkcli infer endpoint get

Get inference endpoint details

## Diagnose from evidence

- With an exact Endpoint ID, call `get` directly. Preserve the entire ID, including `ep-m-`; never invent a suffix or rewrite its prefix. Record actual status, reason, model binding, scope and limits. Use `resources resolve` for ambiguous bindings; a custom model's base-model lineage is not its bound identity.
- Keep the original identity, region, base URL, request time, error code and RequestId. Redact credentials; do not rotate keys, change profiles or widen permissions as a diagnostic shortcut. A 401 requires checking credential shape/validity; a 403 requires checking actual resource access and scope; a 404 requires checking the exact ID and API route. None alone proves a specific cause.
- `Running` is control-plane state, not successful inference. Endpoint usage is aggregated usage, not a single-request error rate or trace. Use Doctor for supported metrics and state missing data or aggregation delay explicitly.
- Only run a minimal real inference when authorized, using the model's supported API and original input modality. For text, use `+chat --model <endpoint-id>`, not `--endpoint`. A text-only probe cannot prove image input works. Without a real call, report the data plane as unverified.
- For 500 errors, inspect API/modality compatibility, parameters and service evidence without assuming every vision model supports embeddings or blaming the network. For ClosedEndpoint, read the status first; starting or rebuilding is a separate authorized write.

## Usage

```bash
arkcli infer endpoint get <endpoint-id> [flags]
```

## Arguments

| Argument | Description | Required |
|----------|-------------|----------|
| `<endpoint-id>` | The ID of the endpoint to get | Yes |

## Flags

| Flag | Type | Description | Required |
|------|------|-------------|----------|
| `-h`, `--help` | | help for get | No |

## Global Flags

| Flag | Type | Description |
|------|------|-------------|
| `--api-key` | string | ARK API key override |
| `--base-url` | string | Custom API base URL |
| `--debug` | | Print request and response debug details to stderr |
| `--format` | string | Output format: json (default "json") |
| `--page-all` | | Automatically fetch all pages when supported |
| `--page-delay` | int | Delay in milliseconds between pages (default 200) |
| `--page-limit` | int | Maximum pages to fetch with --page-all (default 10) |
| `--profile` | string | Active config profile |
| `--transform` | string | Transform output with a GJSON-style path expression |
