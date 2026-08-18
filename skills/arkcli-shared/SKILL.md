---
name: arkcli-shared
version: 0.2.2
description: "Shared Ark CLI execution protocol for BytePlus: first-use setup, authentication gates, command routing, structured output, safety, and confirmation handling. Read this Skill before using any other arkcli-* Skill."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli --help"
---

# Ark CLI shared protocol for BytePlus

Read this file before using any other `arkcli-*` Skill. Keep common execution,
authentication, output, and safety rules here instead of duplicating them in
each capability Skill.

## When To Trigger

- before using any other BytePlus `arkcli-*` Skill;
- on first use, before the owning capability is known;
- when authentication, profile, output, confirmation, or command-routing rules
  are needed by more than one capability.

## When NOT To Trigger

This shared protocol is not the final destination for a capability-specific
task. After applying the common rules, continue with the owning BytePlus Skill.
Do not use this file as a substitute for command-specific flags, output fields,
or recovery guidance.

## References

Load detailed shared guidance only when needed:

| Situation | Reference |
|---|---|
| Global flag or runtime override diagnosis | [`references/global-flags.md`](references/global-flags.md) |
| Default model or Endpoint selection | [`references/profile-defaults.md`](references/profile-defaults.md) |
| Control-plane versus data-plane credential recovery | [`../arkcli-auth/references/auth-modes.md`](../arkcli-auth/references/auth-modes.md) |
| Resolving resources owned by the current identity | [`../arkcli-auth/references/identity-resolution.md`](../arkcli-auth/references/identity-resolution.md) |
| Account-opening and payment verification before billable writes | [`../arkcli-auth/references/realname-gate.md`](../arkcli-auth/references/realname-gate.md) |
| Unclassified command failure | [`references/troubleshooting.md`](references/troubleshooting.md) |

## Product boundary

- This Skill set is for the standalone BytePlus product.
- Do not switch to another tenant or reuse credentials, Regions, endpoints, or
  local state from another product.
- Ark CLI stores BytePlus state under `~/.arkcli-bp/`.
- The control-plane Region is fixed to `ap-southeast-1`; another Region is
  invalid and must not be used as a fallback.
- The control-plane environment is `prod` by default. Use `--env stg` only
  when the user explicitly requests the BytePlus staging environment.
- BytePlus supports the `en_us` UI locale only.
- Route version checks, update notices, and explicit upgrades to
  [`arkcli-update`](../arkcli-update/SKILL.md). An available release never
  authorizes an automatic update.

## Command selection order

Choose commands from the user's goal, not from an API action name.

1. Prefer a product command: `arkcli <domain> <verb>` or
   `arkcli +<workflow>`.
2. Read the capability reference before executing a documented command.
3. Use the API Explorer only when no product command covers the task.
4. Execute read-only operations directly. Confirm user intent before writes,
   deletion, credential rotation, or default-profile changes.

Do not require an Endpoint for a one-off model trial. Use a task workflow such
as chat or generation when that capability is available. Use Endpoint commands
for stable production integration.

## AI Skill invocation protocol

Every `arkcli` command executed by an AI Agent through an `arkcli-*` Skill must
use the complete single-command prefix below. It records the caller and owning
Skill while freezing the installed CLI version for the whole Skill workflow:
no implicit registry refresh, update notice, or automatic apply scheduling may
run between commands.

```bash
ARKCLI_NO_UPDATE_NOTIFIER=1 \
ARKCLI_CALLER_TYPE=ai_agent \
ARKCLI_CALLER_NAME=<agent-id> \
ARKCLI_SKILL_NAME=<current-arkcli-skill> \
arkcli <command> ...
```

- Use a stable Agent ID such as `codex`, `claude-code`, `opencode`, `openclaw`,
  `trae`, or `cursor`; use `unknown_agent` only when it cannot be determined.
- Set `ARKCLI_SKILL_NAME` to the actual owning capability, never
  `arkcli-shared`.
- Apply the full prefix to every `arkcli` command in a multi-command workflow,
  not only the first command.
- Despite its compatibility name, `ARKCLI_NO_UPDATE_NOTIFIER=1` suppresses all
  implicit update activity: cache checks/refreshes, notices, and automatic
  apply scheduling. It does not block an explicit user-requested
  `arkcli update` or `arkcli update --check`.
- Keep the variables scoped to one command; do not export them into unrelated
  shell work.

## Authentication gate

Except for authentication commands, profile inspection, and local Skill
listing, check authentication before a remote business command.

1. Run `arkcli auth status --format json`.
2. If the session is valid, continue the original task.
3. If the session is missing or expired, tell the user that BytePlus SSO is
   starting, then run `arkcli auth login`.
4. In a non-browser or non-interactive environment, use the two-phase flow:

```bash
arkcli auth login --no-browser
arkcli auth login --no-browser --code <authorization-code>
```

5. After login succeeds, resume the original task. Authentication is an
   intermediate step, not the final result.

Do not print, persist, or echo full tokens, access keys, secret keys, or API
keys.

## Account verification gate

Before model activation, deployment, inference Endpoint creation, fine-tuning
creation, or Managed Agent creation, run:

```bash
arkcli auth status --format json
```

Read `byteplus_sso.identity.verified` before executing one of these billable
writes:

- `true`: continue.
- `false`: stop, send the user to
  `https://console.byteplus.com/user/basics/`, and wait for the user to finish
  both account-opening and payment verification.
- absent: the result is unknown; continue without claiming either state and
  preserve any structured backend error.

See
[`../arkcli-auth/references/realname-gate.md`](../arkcli-auth/references/realname-gate.md)
for the exact two-check contract and failure rules.

## Output contract

- Prefer `--format json` or `--format yaml` for automation.
- Use `--transform` when the next step needs a stable field rather than the
  full response.
- Treat stdout as structured business output. Send explanations, diagnostics,
  and errors to stderr.
- Do not parse decorative or table-formatted text when structured output is
  available.

## Confirmation handling

For destructive, credential-changing, or cost-sensitive operations:

1. Confirm the exact target, scope, and cost or destructive impact before
   adding `--yes` or otherwise authorizing the write.
2. If Ark CLI returns `type="requires_confirmation"`, treat it as a normal
   interaction boundary and show the action and its
   impact to the user.
3. After confirmation, repeat the same command with `--yes`.
4. If the user declines, stop without changing state.

## Guard Checklist

- Do not use the API Explorer as the default entry point.
- Do not retry business commands repeatedly before checking authentication and
  configuration.
- Do not treat login, model lookup, or profile selection as the user's final
  outcome when the original task still has remaining steps.
- Do not invent invocation-global `--project-name` or `--region` overrides.
  Select or update a profile for runtime project and Region scope. Use a
  command-local project/Region flag only when that exact command documents it
  for its own resource or query semantics.
- Do not infer BytePlus support from a similarly named command or Skill in
  another product. Use the installed BytePlus command help and the matching
  BytePlus capability Skill.
