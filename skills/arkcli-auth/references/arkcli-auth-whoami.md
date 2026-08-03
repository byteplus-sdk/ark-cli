# `auth whoami`

Use `whoami` when a task depends on the active identity rather than only on
credential health.

| Command | Primary question |
|---|---|
| `arkcli auth status --format json` | Are credentials healthy and which runtime scope is effective? |
| `arkcli auth whoami --format json` | Which user and account are active? |

## Commands

```bash
arkcli auth whoami --format json
arkcli auth whoami --transform user_id
arkcli auth whoami --transform name
```

The exact field set depends on the identity claims returned by BytePlus. When
available, common fields include:

```json
{
  "logged_in": true,
  "auth_method": "sso",
  "name": "alice",
  "user_id": "10000001",
  "account_id": "2000000001",
  "trn": "trn:iam::2000000001:user/alice",
  "is_root": false,
  "project_name": "default",
  "region": "ap-southeast-1"
}
```

Do not assume every optional identity field is present. Use only fields
returned by the current command.

## Agent rules

- Do not decode JWT or token files under `~/.arkcli-bp/`; consume the command's
  structured output.
- Do not confuse account-level and user-level identifiers.
- Do not invent a universal `my resources` filter. Each resource Skill owns
  the mapping from identity fields to resource filters or tags.
- If `logged_in=false` or `sso_expired=true`, run the login flow and then resume
  the original task.
- If a required identity field is absent after successful SSO, report that the
  current identity contract cannot satisfy the requested filter. Do not fall
  back to another product or account.

## Expired session shape

An expired session may still expose non-secret historical identity fields for
diagnosis while setting `logged_in=false`. Treat those fields as context only;
do not use them to authorize a remote operation until login succeeds again.
