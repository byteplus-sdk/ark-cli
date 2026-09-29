# arkcli infer endpoint list

List inference endpoints

## Inventory and export

- Preserve the requested identity and filters. An empty personal query does not authorize an account-wide query. For an explicitly account-wide running inventory, use `--status Running --page-all --page-size 100`.
- Parse complete stdout separately from stderr. The array is `Items`, not `items`; check pagination limits and warnings before calling the inventory complete. Never use `head` as a completeness check.
- Include the exact ID, name, actual bound model, status, creation time and description. Fetch per-Endpoint details only for requested fields absent from the list; distinguish missing values from zero.
- Fetch details serially by default and respect API QPS. Record partial failures individually; do not mix `2>&1` into JSON or claim full success after failures. Quote IDs as arguments rather than inserting returned values into shell source or file paths.
- Preserve raw timestamps, label the display timezone and convert offsets only once. Report query scope, filters, count, failures and completeness alongside the export.

## Usage

```bash
arkcli infer endpoint list [flags]
```

## Flags

| Flag | Type | Description | Required |
|------|------|-------------|----------|
| `--format` | string | Output format: table \| json | No |
| `--model` | string | Filter by model: a custom model ID (`cm-...`) or a foundation model name (`doubao-seedream-5-0-pro`, or the dotted DisplayName form `doubao-seedream-5.0-pro`) | No |
| `--status` | string | Filter by endpoint status | No |
| `--mine` | bool | List only endpoints created by the current SSO sub-user (server-side `sys:ark:createdBy` tag filtering) | No |
| `--page-number` | int | Page number (>=1) | No |
| `--page-size` | int | Page size | No |
| `-h`, `--help` | | help for list | No |

## `--model` semantics

`--model` accepts two kinds of value, dispatched by the CLI:

- `cm-` prefix → filtered as a **custom model ID** (`Filter.CustomModelIds`, case-sensitive).
- anything else → filtered as a **foundation model name**: the value is normalized to
  the connector Name first, then sent as `Filter.FoundationModelNames` (exact match
  server-side).

Normalization does exactly two things and **never rewrites characters**: an exact Name
match, then a case-insensitive exact DisplayName match. DisplayName is not always the
same shape as Name — for example the model `doubao-seedream-5-0` has DisplayName
`Doubao-Seedream-5.0-lite` (the display name carries `lite`, the connector Name does
not), so replacing dots with hyphens would produce a name that does not exist.

If the value is neither a `cm-` ID nor a match for any foundation model name, the
command **exits non-zero** with a hint instead of returning an empty list. Do not read
an empty result as "this account has no endpoint for that model".

```bash
arkcli infer endpoint list --model doubao-seedream-5-0-pro --page-all --page-size 100 --format json
```

## Semantics of "my endpoints" (overrides shared default)

This command **has built-in `--mine` support**. According to the "decision order" in [`../../arkcli-auth/references/identity-resolution.md`](../../arkcli-auth/references/identity-resolution.md), you MUST **prefer** this option and must not fall back to client-side filtering with `whoami + jq`.

```bash
arkcli infer endpoint list --mine --page-all --page-size 100 --page-delay 800 --format json
```

Behavior:

- At the shortcut layer, `--mine` combines the current `cfg.UserID` / `cfg.UserName` into `IAMUser/<UserID>/<UserName>` and sends it to `ListEndpoints` as `TagFilters.{Key: "sys:ark:createdBy", Values: [...]}`.
- It can be freely combined with `--status` / `--model` / `--page-*` (for example, `--mine --status Running`).
- It is available only for **SSO sub-user** login state. Other states **fail directly** and do not silently degrade:
  - Non-SSO (AK/SK / APIKey): `--mine requires an SSO sub-user login`; run `arkcli auth login` to re-establish BytePlus identity.
  - SSO root: `--mine is not supported for root logins`; guide the user to log in with a sub-user.
  - SSO with missing UserID/UserName claims: guide the user to log in again to refresh identity.

## Global Flags

| Flag | Type | Description |
|------|------|-------------|
| `--api-key` | string | ARK API key override |
| `--base-url` | string | Custom API base URL |
| `--debug` | | Print request and response debug details to stderr |
| `--page-all` | | Automatically fetch all pages when supported |
| `--page-delay` | int | Delay in milliseconds between pages (default 200) |
| `--page-limit` | int | Maximum pages to fetch with --page-all (default 10) |
| `--profile` | string | Active config profile |
| `--transform` | string | Transform output with a GJSON-style path expression |
