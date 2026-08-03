---
name: arkcli-auth
version: 0.2.0
description: "Manage Ark CLI authentication for BytePlus: browser SSO, two-phase no-browser SSO, status inspection, identity inspection, API key selection, and logout. Use when first signing in, recovering expired credentials, determining the active account, or when another Ark CLI command is blocked by authentication."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli auth --help"
---

# Ark CLI authentication for BytePlus

Before using this Skill, read
[`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md). Authentication is a
gate for the user's original task, not an outcome by itself.
The entire authentication domain is an identity/TTY workflow and rejects
`--dry-run`; never generate that flag.

## References

Load command details only when needed:

| Situation | Reference |
|---|---|
| Browser and two-phase no-browser login | [`references/arkcli-auth-login.md`](references/arkcli-auth-login.md) |
| Credential health, project/Region diagnosis, and logout | [`references/arkcli-auth-status.md`](references/arkcli-auth-status.md) |
| Active user and account identity | [`references/arkcli-auth-whoami.md`](references/arkcli-auth-whoami.md) |
| Control-plane versus data-plane credential recovery | [`references/auth-modes.md`](references/auth-modes.md) |
| Resolving a request for resources owned by the current identity | [`references/identity-resolution.md`](references/identity-resolution.md) |
| Account-opening and payment verification before billable writes | [`references/realname-gate.md`](references/realname-gate.md) |
| Maintainer regression cases | [`references/evals.md`](references/evals.md) |

## When To Trigger

- first-time BytePlus SSO login;
- expired or missing control-plane credentials;
- `who am I`, active account, or active identity questions;
- ARK API key selection or recovery;
- an explicit request to log out;
- a business command that failed because authentication is not ready.

## When NOT To Trigger

Do not use this Skill to decide whether a model, Endpoint, or business feature
is supported. Route those questions to the matching BytePlus capability Skill.
Do not run `auth apikey` when the user only wants a read-only key inventory; use
`arkcli profile keys list`.

## Execution order

1. Run `arkcli auth status --format json`.
2. If `logged_in` is true and the required credential is healthy, resume the
   original task.
3. If login is required, tell the user that BytePlus SSO is starting and run
   `arkcli auth login` with a generous interactive timeout.
4. After a successful login, immediately resume the original task.
5. Run `arkcli auth logout` only after the user explicitly asks to clear local
   credentials.

Use `arkcli auth whoami --format json` when the request depends on the current
identity. Do not decode token files or read `~/.arkcli-bp/` directly.

## Browser SSO

For a normal interactive terminal:

```bash
arkcli auth login
```

The standalone product selects BytePlus SSO at compile time. Do not add a
tenant selector or a product-specific subcommand.

If browser launch, callback binding, or authorization times out, do not loop.
Return stderr to the user and offer the no-browser flow.

## No-browser SSO

In automation, a sandbox, a remote host, or another non-interactive
environment, use the two-phase flow.

Phase 1:

```bash
arkcli auth login --no-browser
```

The command returns structured output containing `stage="authorize_pending"`,
an `authorize_url`, and the next command. Forward the URL unchanged. Ask the
user to finish authorization in any browser and return the displayed base64
authorization code.

Phase 2:

```bash
arkcli auth login --no-browser --code <authorization-code>
```

Both phases must use the same `HOME` and persistent filesystem because Phase 2
consumes the pending session created by Phase 1 under `~/.arkcli-bp/`.

Recovery rules:

- missing or expired pending session: rerun Phase 1;
- malformed code or state mismatch: verify that the code belongs to the most
  recent authorization URL, then retry Phase 2 within the session TTL;
- transient token exchange failure: retry Phase 2 only when the command marks
  the operation retryable;
- never print or store the authorization code outside the active command.

## API key handling

`arkcli auth apikey` is an interactive selection command and changes the
active key. Do not run it when the user only wants to list keys.

- Inspect the current selected key through the masked `ark_api_key` field in
  `arkcli auth status --format json`.
- Use profile key list commands for read-only inventory.
- Never print a complete API key.
- Do not confuse an ARK API key used by data-plane calls with the SSO/STS
  credentials used by control-plane calls.

## Logout

Logout is destructive because it clears the local BytePlus identity state.
Run it only after explicit user intent:

```bash
arkcli auth logout
```

After logout, verify status with:

```bash
arkcli auth status --format json
```

## Command summary

| Command | Purpose |
|---|---|
| `arkcli auth status --format json` | Inspect credential health and effective identity context. |
| `arkcli auth whoami --format json` | Inspect the active user and account identity. |
| `arkcli auth login` | Start browser SSO. |
| `arkcli auth login --no-browser` | Start Phase 1 of cross-device SSO. |
| `arkcli auth login --no-browser --code <code>` | Complete Phase 2 of cross-device SSO. |
| `arkcli auth apikey` | Interactively select and persist an ARK API key. |
| `arkcli auth logout` | Clear local credentials after explicit confirmation. |

## Guard Checklist

- Never reveal complete tokens, access keys, secret keys, API keys, or pending
  authorization codes.
- Do not treat an authorization failure as proof that a capability is absent.
- Do not reuse credentials or local state from another product.
- Do not stop after login when the user's original task is still incomplete.
