# Authentication Boundaries and Recovery

> Read this reference when a command fails because a control-plane identity or a data-plane API key is missing, expired, or unauthorized.

## Two Authentication Planes

BytePlus Ark CLI uses different credentials for the control plane and the data plane:

| Plane | Typical operations | Credential |
|---|---|---|
| Control plane | List models, manage endpoints, query API keys, deploy resources | Temporary STS credentials derived from BytePlus SSO |
| Data plane | Chat, generation, embeddings, multimodal inference | Ark API key sent as a bearer token |

Do not substitute one credential for the other. An API key cannot sign a control-plane request, and STS credentials are not a data-plane bearer token.

## Control-Plane Recovery

1. Run `arkcli auth status --format json`.
2. If the CLI reports that SSO is missing, expired, or cannot be refreshed, run:

   ```bash
   arkcli auth login
   ```

3. Retry the original command once.
4. If the command still fails with a permission error, treat it as a capability or account-permission issue. Do not repeatedly log in or switch to another product identity.

The CLI stores the refreshable BytePlus identity and STS cache under `~/.arkcli-bp/`. Never read or edit those files directly; use `auth status`, `auth whoami`, `auth login`, and `auth logout`.

## Data-Plane Recovery

Typical data-plane authentication failures include HTTP 401/403, `InvalidApiKey`, `Unauthorized`, `AuthenticationError`, and messages that mention an expired or missing API key.

Use this recovery sequence:

1. Check whether `ARK_API_KEY` or `--api-key` overrides the active profile. An override takes precedence over stored profile data.
2. If the key comes from the active profile, run:

   ```bash
   arkcli profile keys refresh
   ```

3. Retry the original command once.
4. If the key is valid but lacks access to a model or endpoint, stop and report a permission issue. Refreshing the same key does not grant new permissions.
5. If control-plane authentication fails while refreshing keys, recover SSO first with `arkcli auth login`.

`profile keys refresh` synchronizes existing backend keys; it does not rotate a
key or grant access. Use the BytePlus Coding Plan references for explicit key
rotation or team-seat management.

## Rules for Agents

- Prefer SSO-derived STS for BytePlus control-plane commands.
- Never expose tokens, STS credentials, API keys, or local identity-store contents.
- Never fall back to another product or tenant identity.
- Retry a failed operation at most once after the relevant credential has been recovered.
- Keep structured output on stdout and recovery guidance on stderr.
