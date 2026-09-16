# `auth login`

Use browser SSO for an interactive workstation:

```bash
arkcli auth login
```

The standalone binary already selects BytePlus. Do not append a tenant
argument or use a command copied from another product.

## Two-phase no-browser flow

Use this flow for agents, sandboxes, CI jobs, remote hosts, and any environment
without a usable local browser.

### Phase 1

```bash
arkcli auth login --no-browser
```

In a non-interactive process, the command creates a short-lived pending SSO
session under `~/.arkcli-bp/`, emits structured output, and exits without
waiting on stdin.

Example shape:

```json
{
  "stage": "authorize_pending",
  "method": "sso_no_browser",
  "authorize_url": "https://signin.byteplus.com/...",
  "next_command": "arkcli auth login --no-browser --code <authorization-code>",
  "expires_in_sec": 600
}
```

Forward `authorize_url` exactly as returned. Do not decode, normalize, or
rewrite it. Ask the user to authorize in any browser and return the base64 code
shown by the page.

### Phase 2

```bash
arkcli auth login --no-browser --code <authorization-code>
```

`--code` is valid only together with `--no-browser`. Phase 2 reads the PKCE and
state values saved by Phase 1, exchanges the code, activates the identity, and
removes the pending session after success.

Both phases must run with the same `HOME` and persistent filesystem. Do not
dispatch them to different containers, workers, or home directories.

## Recovery table

| Failure | Recovery |
|---|---|
| Pending session missing or expired | Rerun Phase 1 and use the new URL. |
| Authorization code is not valid base64 | Correct the code and retry Phase 2 while the pending session remains valid. |
| State mismatch | Use the code produced by the latest URL; rerun Phase 1 if provenance is uncertain. |
| Retryable token exchange failure | Retry Phase 2 within the pending-session TTL. |
| Browser launch or callback failure | Switch to the no-browser flow instead of looping browser login. |

Never log the complete authorization code, token response, API key, access key,
or secret key.

## API-key provisioning after login

Login resolves API keys by profile type. `platform` and personal `coding-plan`
profiles use the ordinary API-key pool; `coding-plan-team` profiles use the key
returned by a Running seat through `GetSeatInfo.Result.ApiKey`.

When a real TTY login finds the ordinary pool empty, it asks whether to create
one all-resource key. Confirmation permits exactly one `CreateApiKey` mutation
and at most three subsequent status reads; the first Active result ends the
check. The current concrete project is used, while an empty or account-wide
scope falls back to `default`; the generated name is
`api-key-YYYYMMDDHHmmss`. Declining or using a non-interactive process performs
no mutation.

`arkcli auth apikey` only selects from the ordinary pool. It cannot obtain or
repair a Coding Plan Team seat key.

For an explicit request to provision an ordinary key, use
`arkcli auth apikey create`. A TTY asks for confirmation; in a non-interactive
environment, add `--yes` only after the end user explicitly authorizes the
write. The command creates at most once, performs no more than three read-only
status checks, saves the verified key for the current identity, and never
prints the raw key. Use `profile keys refresh/use` to change an existing
Profile's default key.
