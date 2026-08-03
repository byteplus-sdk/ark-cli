# arkcli infer endpoint create

Create an inference endpoint.

## Usage

```bash
arkcli infer endpoint create [flags]
```

## Flags

| Flag | Type | Description | Required |
|------|------|-------------|----------|
| `--model` | string | Model ID (foundation model or custom model) | Yes |
| `--name` | string | Endpoint name | Yes |
| `--billing-method` | string | Billing / inference method. Currently supports only `token`. If omitted, the default behavior is used. | No |
| `--rpm` | int | Rate limit RPM | No |
| `--tpm` | int | Rate limit TPM | No |
| `--dry-run` | bool | Emit the local create plan without model discovery, activation, or create API calls | No |
| `-h`, `--help` | | help for create | No |

Model resolution and activation checks require online state, so Client Preview
uses `fidelity=partial` and lists them under `unresolved`. It is not server
validation and does not prove that the model is activated or the request will
be accepted.

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

## Examples

```bash
# Create an Endpoint
arkcli infer endpoint create \
  --model dola-seed-2-1-turbo-260628 \
  --name my-endpoint

# Explicitly request the token inference method (currently keeps the same create payload as when omitted, but first checks whether the model supports token)
arkcli infer endpoint create \
  --model dola-seed-2-1-turbo-260628 \
  --name my-token-endpoint \
  --billing-method token

# Typical response
{
  "Id": "ep-20260421180049-ngwkm"
}

# Extract only the Id for reuse by subsequent commands
endpoint_id=$(arkcli infer endpoint create \
  --model dola-seed-2-1-turbo-260628 \
  --name my-endpoint \
  --transform Id)
```

## Return value

After creation succeeds, the command returns the `Id` of the new Endpoint. Example:

```json
{
  "Id": "ep-20260421180049-ngwkm"
}
```

This `Id` is the primary entry point for subsequent operations.

## Next steps after creation

### 1. View details

```bash
arkcli infer endpoint get "$endpoint_id"
```

### 2. Manage the lifecycle

```bash
arkcli infer endpoint stop "$endpoint_id"
arkcli infer endpoint start "$endpoint_id"
```

### 3. Do not call `+deploy` again

If you already obtained an `Id` through `infer endpoint create`, the Endpoint has already been created.

At this point, do not run:

```bash
arkcli +deploy ...
```

Because `+deploy` is itself a "create Endpoint" workflow. It creates a new resource instead of reusing the `Id` that was just created.

## Notes

- `infer endpoint create` is oriented toward standard resource creation.
- `+deploy` is oriented toward the task workflow of "create Endpoint + model activation / reuse check / profile sync".
- Both commands create Endpoints and should not be executed sequentially as duplicate steps.
- Voice models (TTS / ASR / dubbing / podcast / voice / real-time voice interaction) cannot currently use this command to create Endpoints. Voice models are not supported.
- `--billing-method` currently has only one enum value: `token`. Passing `token` does not add extra fields to the CreateEndpoint request. It only checks before creation whether the corresponding model supports token-based inference.
- Foundation model support methods come from `ArkModels.data[].Features.ShareService`. For custom models, prefer the deployable version's `EndpointSupportedMethods.ShareService`, and fall back to `AvailableDeploymentTypes=Shared` when necessary.

## References

- [arkcli-infer-endpoint](../SKILL.md) -- infer endpoint capability overview
- [arkcli-infer-endpoint-get](arkcli-infer-endpoint-get.md) -- View Endpoint details
- [arkcli-infer-endpoint-start](arkcli-infer-endpoint-start.md) -- Start an Endpoint
- [arkcli-infer-endpoint-stop](arkcli-infer-endpoint-stop.md) -- Stop an Endpoint
