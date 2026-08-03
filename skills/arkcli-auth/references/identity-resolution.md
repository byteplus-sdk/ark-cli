# Resolving “My Resources”

> Read this reference when the user asks for resources created, owned, or used by the current BytePlus identity.

“My resources” must be resolved against the active SSO identity. Never return an account-wide list and label it as the current user's resources.

## Decision Order

1. If the capability command provides `--mine` or another documented ownership filter, use it.
2. Otherwise, read that capability's reference and use its documented server-side ownership field or tag filter.
3. If no ownership rule is documented, inspect a small read-only response and report that the capability still needs a BP ownership audit. Do not reuse a filter from another resource type.

Ownership is resource-specific. Endpoints, API keys, models, usage records, and managed agents may expose different ownership fields, or may only support account-level scope.

## Identity Inspection

Use public CLI commands only:

```bash
arkcli auth whoami --format json
arkcli auth status --format json
```

If SSO is expired or the identity is unavailable, ask the user to run `arkcli auth login` before applying a personal ownership filter.

## Rules for Agents

- Do not read or decode files under `~/.arkcli-bp/`.
- Do not ask the user to paste a user ID, token, or STS credential.
- Do not use `whoami` plus a client-side filter when the command already supports `--mine`.
- Do not assume that an ownership filter verified for another product is valid for BytePlus.
- Document the ownership rule in the capability-specific reference after the BP audit confirms it.
- For shell variables, avoid reserved names such as `UID`, `EUID`, `HOME`, `PATH`, `PWD`, and `USER`.
