# Global flags

Read this reference when building a complex command or diagnosing profile,
data-plane, environment, or output overrides. Command-specific flags remain
authoritative in `arkcli <domain> <verb> --help`.

| Flag | Purpose |
|---|---|
| `--profile` | Select the active profile for this command. |
| `--api-key` | Override the ARK API key for this command. |
| `--base-url` | Override the data-plane base URL. |
| `--env` | Select the BytePlus control-plane environment: `prod` or `stg`. |
| `--lang` | Select the UI locale. BytePlus accepts `en_us` only. |
| `--ui-lang` | Unambiguous alias for the global UI locale when a command has a historical local `--lang` flag. BytePlus accepts `en_us` only. |
| `--format` | Select `json`, `yaml`, `table`, `csv`, `jsonl`, or `pretty`. Automation should prefer `json` or `yaml`. |
| `--transform` | Extract a stable field or subtree from structured output. |
| `--page-all` | Follow all available pages. |
| `--page-limit` | Limit automatic pagination. |
| `--page-delay` | Set the delay between pages in milliseconds. |
| `--debug` | Send diagnostic details to stderr. |

`--dry-run` is not a global flag. Only a leaf command with Client Preview
support registers it; verify the leaf command's `--help`. A supported preview
emits local `preview.v1` output without network access, filesystem writes, or
subprocesses. An unsupported leaf rejects the flag explicitly.

## Override diagnosis

When persisted profile values and runtime behavior differ:

1. Run `arkcli auth status --format json` to inspect effective runtime values.
2. Run `arkcli profile show --format json` to inspect persisted profile data.
3. Check supported invocation flags and environment variables before changing
   the profile. `--region`, `--project-name`, `ARK_REGION`, and
   `ARK_PROJECT_NAME` are not runtime override mechanisms.
4. Keep BytePlus local state under `~/.arkcli-bp/`; do not inspect or modify a
   different product's state directory.

## Important precedence rules

```text
profile:
  --profile > ARK_PROFILE > config default_profile
  > first platform profile > default sentinel

project:
  active profile project > bound identity fallback > default

control-plane environment:
  --env > ARK_CLIENT_ENV > active profile env > prod

Region:
  active profile region > BytePlus fixed ap-southeast-1

temporary data-plane API key:
  --api-key > ARK_API_KEY > active profile > bound identity

temporary data-plane base URL:
  --base-url > ARK_BASE_URL > active profile
  > product/profile derivation > BytePlus default
```

An explicit unsupported environment, Region, or locale is an input error. Do
not silently replace it with another product or Region.

`--base-url` affects the data plane. `--env` selects the BytePlus control-plane
environment. Do not use one as a substitute for the other.

An explicit `--base-url` or `ARK_BASE_URL` must be accompanied by an explicit
`--api-key` or `ARK_API_KEY`. This prevents a profile API key from being sent
to an overridden endpoint. For `+chat`, `+understand`, and `+gen`, an explicit
API key plus an endpoint ID may derive the missing BytePlus base URL from
authoritative endpoint metadata.
