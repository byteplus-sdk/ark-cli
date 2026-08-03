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
