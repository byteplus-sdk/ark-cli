# arkcli infer endpoint list

List inference endpoints

## Usage

```bash
arkcli infer endpoint list [flags]
```

## Flags

| Flag | Type | Description | Required |
|------|------|-------------|----------|
| `--format` | string | Output format: table \| json | No |
| `--model` | string | Filter by model ID (custom model) | No |
| `--status` | string | Filter by endpoint status | No |
| `--mine` | bool | List only endpoints created by the current SSO sub-user (server-side `sys:ark:createdBy` tag filtering) | No |
| `--page-number` | int | Page number (>=1) | No |
| `--page-size` | int | Page size | No |
| `-h`, `--help` | | help for list | No |

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
