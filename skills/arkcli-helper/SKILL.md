---
name: arkcli-helper
version: 0.1.1
description: "Configure an IDE, editor, or supported local coding agent for BytePlus Platform, Coding Plan, or Coding Plan Team: use arkcli helper configure to connect Claude Code, Codex, OpenCode, OpenClaw, Hermes, or Pi to a model/provider or Endpoint, helper list to inspect support, and helper reset for safe rollback. Prefer arkcli-helper over arkcli-auth or arkcli-config when the user's goal is wiring a model into a local coding client, even if profile or API-key setup is also mentioned. MCP injection and unsupported clients are outside the BytePlus Helper surface."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli helper --help"
---

# arkcli helper

Read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) before running a
command. This Skill covers the BytePlus client configuration surface only.

## When To Trigger

Use this Skill when the user wants to:

- configure an IDE, editor, or coding agent for BytePlus, even when the user
  does not name the exact harness yet;
- add, install, connect, or wire a model/provider into a local coding client;
- discover which local AI coding clients BytePlus Helper supports;
- configure Claude Code, Codex, OpenCode, OpenClaw, Hermes, or Pi;
- connect a client to a BytePlus Platform Endpoint;
- configure a client from a `coding-plan` or `coding-plan-team` profile;
- inspect or remove configuration previously managed by arkcli;
- choose Codex profile scope versus global scope.

## When NOT To Trigger

- Use [`arkcli-auth`](../arkcli-auth/SKILL.md) for login, logout, identity, or
  API key selection.
- Use [`arkcli-profile`](../arkcli-profile/SKILL.md) to inspect, create, or
  switch profiles.
- Use [`arkcli-connect`](../arkcli-connect/SKILL.md) to install arkcli Skills
  into local agents. It does not configure model providers.
- Use [`arkcli-infer-endpoint`](../arkcli-infer-endpoint/SKILL.md) to create,
  start, or manage Platform Endpoints.
- Do not route MCP injection, database add-on setup, or unrelated subscription
  families through BytePlus Helper. Do not fall back to another product or
  tenant.

If a prompt mentions authentication or profiles only as prerequisites for
configuring an IDE, editor, or coding agent, keep the primary route on this
Skill and use `arkcli-auth` or `arkcli-profile` only for the blocking recovery
step. Generic local configuration with no client-integration goal belongs to
`arkcli-config`.

## Product Boundary

Supported profile types:

- `platform`
- `coding-plan`
- `coding-plan-team`

Supported harness IDs:

- `claude-code`
- `codex`
- `opencode`
- `openclaw`
- `hermes`
- `pi`

Trae is not accepted because its current integration cannot write model
provider configuration. Always use `arkcli helper list` as the runtime source
of truth.

A Coding Plan model may be the routing model `ark-code-latest`. Switching its
route target (Auto smart scheduling or a pinned underlying model) is outside
this skill — use `arkcli plans model-apply --plan <plan> --model <target>`
(see [`../arkcli-plans/`](../arkcli-plans/SKILL.md)); it writes the control
plane and stays in sync with the console.

## Command Map

| Command | Mode | Purpose |
|---|---|---|
| `arkcli helper` | Interactive, writes after confirmation | Select profile, Endpoint or model, and client in a TTY wizard. It may offer to install a missing client. |
| `arkcli helper list` | Read-only | List supported clients, installation state, and settings paths. No login is required. |
| `arkcli helper configure <harness>` | Non-interactive write | Write model/provider configuration for one supported client. It does not install the client. |
| `arkcli helper reset <harness>` | Non-interactive write | Remove only arkcli-managed model/provider configuration and preserve unrelated settings. |

## Execution Order

1. Run `arkcli helper list` and the relevant `--help` command first.
2. Determine the exact harness, profile type, profile name, and model or
   Endpoint.
3. For a Platform profile, verify that the Endpoint is owned by the current
   SSO sub-user, Running, and text-output capable.
4. For Codex, decide whether to use the default `arkcli` profile scope or the
   explicitly requested global scope.
5. Before `configure` or `reset`, show the exact command and affected settings
   files, then obtain explicit user confirmation.
6. Run the command, report its result, and explain any required client reload
   or Codex profile activation.

```text
request
  |
  +-- inspect only ----------> helper list
  |
  +-- guided setup in TTY ---> helper
  |
  +-- exact target known ----> helper configure <harness>
  |
  +-- remove managed config -> helper reset <harness>
```

## Configuration Contract

- `platform`: `--endpoint <endpoint-id>` is required and `--model` is invalid.
- `coding-plan` or `coding-plan-team`: `--endpoint` is invalid. `--model` is
  optional; when omitted, arkcli uses the selected or first available plan
  model.
- `--profile` defaults to the active profile, but explicit profile selection is
  safer for automated or non-interactive work.
- Plan configuration uses the API key stored with the selected profile.
  One-off API key overrides are not a substitute for a missing profile key.
- `configure` writes immediately and does not install a missing client.
- `reset` writes immediately and removes only arkcli-managed entries.

## Codex Scope

The default Codex scope is the named profile `arkcli`:

```bash
arkcli helper configure codex \
  --profile <profile-name> \
  --model <model-id>
```

This writes `$HOME/.codex/arkcli.config.toml`. Start Codex with:

```bash
codex --profile arkcli
```

Use `--codex-config-scope global` only when the user explicitly wants
`$HOME/.codex/config.toml` changed. A custom `--codex-profile` must contain only
letters, digits, underscores, or hyphens.

## Guard Checklist

- [ ] The requested profile is `platform`, `coding-plan`, or
      `coding-plan-team`.
- [ ] The harness appears in `arkcli helper list`.
- [ ] Platform uses `--endpoint`; plan profiles use optional `--model`.
- [ ] The target profile, model or Endpoint, Codex scope, and settings paths
      have been shown to the user.
- [ ] The user explicitly approved any `configure` or `reset` command.
- [ ] Never add `--dry-run` to a Helper command. The entire `arkcli helper`
      domain rejects it as an unknown flag because Helper is a TTY/local
      configuration workflow, not API request preview.
- [ ] No command, flag, profile, or fallback from another product is suggested.
- [ ] Completion reports the command result and any reload or activation step.

## Recovery Routing

- Not logged in or session expired: follow
  [`arkcli-auth`](../arkcli-auth/SKILL.md), then retry.
- Wrong or unsupported profile: inspect and switch with
  [`arkcli-profile`](../arkcli-profile/SKILL.md).
- Missing stored API key: run `arkcli auth apikey` or
  `arkcli profile keys refresh`, then retry.
- No eligible Platform Endpoint: create one with
  `arkcli infer endpoint create`; start a stopped one with
  `arkcli infer endpoint start <endpoint-id>`.
- Parent wizard used without a TTY: use the explicit
  `arkcli helper configure <harness> ...` form.

## References

- Read [`references/arkcli-helper.md`](references/arkcli-helper.md) for exact
  command grammar, client paths, Endpoint eligibility, errors, and recovery.
- Read [`references/evals.md`](references/evals.md) for trigger, anti-trigger,
  guard, and happy-path evaluation cases.
