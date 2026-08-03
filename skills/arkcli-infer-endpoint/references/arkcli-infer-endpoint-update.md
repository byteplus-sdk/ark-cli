# arkcli infer endpoint update

Update an inference endpoint (name / description / rate limits)

## Usage

```bash
arkcli infer endpoint update <endpoint-id> [flags]
```

At least one of `--name`, `--description`, `--rpm`+`--tpm`, or any `--cg-*` option must be provided. Otherwise, the command reports "nothing to update" and does not send a request.

## Arguments

| Argument | Description | Required |
|----------|-------------|----------|
| `<endpoint-id>` | The ID of the endpoint to update | Yes |

## Flags

| Flag | Type | Description | Required |
|------|------|-------------|----------|
| `--name` | string | New endpoint name | No |
| `--description` | string | New endpoint description (use empty string `""` to clear) | No |
| `--rpm` | int | Rate limit: requests per minute (must be paired with `--tpm`) | No |
| `--tpm` | int | Rate limit: tokens per minute (must be paired with `--rpm`) | No |
| `--cg-concurrent-requests` | int | ContentGeneration: max concurrent requests (image-generation endpoints) | No |
| `--cg-create-task-rpm` | int | ContentGeneration: CreateTask RPM (image-generation endpoints) | No |
| `--dry-run` | bool | Emit a local `UpdateEndpoint` Client Preview without calling the API | No |
| `-h`, `--help` | | help for update | No |

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

Rename only:

```bash
arkcli infer endpoint update ep-20260428194457-p8b47 --name seed-2-1-turbo-demo-123
```

Clear the description:

```bash
arkcli infer endpoint update ep-20260428194457-p8b47 --description ""
```

Adjust rate limits (`--rpm` and `--tpm` must be provided together):

```bash
arkcli infer endpoint update ep-20260428194457-p8b47 --rpm 60 --tpm 100000
```

Adjust content generation parameters for image-generation endpoints (valid only for image-generation scenarios):

```bash
arkcli infer endpoint update ep-20260507230659-skvgn \
  --cg-concurrent-requests 10 \
  --cg-create-task-rpm 60
```

Local preview (does not send a request):

```bash
arkcli infer endpoint update ep-20260428194457-p8b47 --name foo --dry-run
```
