# `arkcli helper` Reference

Read [`../SKILL.md`](../SKILL.md) first. This reference documents the BytePlus
Helper surface and its local configuration effects.

## Exact Command Grammar

```bash
arkcli helper
arkcli helper list
arkcli helper configure <harness> \
  [--profile <profile-name>] \
  [--model <model-id>] \
  [--endpoint <endpoint-id>] \
  [--codex-config-scope profile|global] \
  [--codex-profile <profile-name>]
arkcli helper reset <harness> \
  [--codex-config-scope profile|global] \
  [--codex-profile <profile-name>]
```

Supported harness IDs are `claude-code`, `codex`, `opencode`, `openclaw`, and
`hermes`. Verify them at runtime instead of assuming:

```bash
arkcli helper list
```

## Read-only Inspection

The following commands do not modify client configuration:

```bash
arkcli helper --help
arkcli helper configure --help
arkcli helper reset --help
arkcli helper list
```

`arkcli helper list` reads the local filesystem and does not require login. It
shows supported clients, whether each client is installed, and the settings
path that Helper knows about.

## Interactive Wizard

Run the parent command in a TTY when the user wants guided selection:

```bash
arkcli helper
```

The wizard can:

1. ensure the user is logged in;
2. select a supported profile;
3. select an eligible Platform Endpoint or Coding Plan model;
4. select a supported client;
5. offer to install a missing client;
6. write the chosen configuration after confirmation.

Without a TTY, the wizard cannot prompt. Use the explicit `configure` form
instead.

## Non-interactive Configure

`configure` is appropriate only when the exact target is already known:

```bash
arkcli helper configure <harness> \
  --profile <profile-name> \
  --model <model-id>
```

It writes configuration immediately. It does not install a missing client.
Before running it:

1. inspect `arkcli helper list`;
2. show the exact harness, profile, model or Endpoint, scope, and settings
   files;
3. obtain explicit user confirmation.

The entire Helper command domain rejects `--dry-run` as an unknown flag.
Helper configures local harness state and is not an API request-preview
surface. Do not generate either `arkcli --dry-run helper ...` or
`arkcli helper ... --dry-run`.

## Coding Plan Profiles

Accepted plan profile types are:

- `coding-plan`
- `coding-plan-team`

Example:

```bash
arkcli helper configure opencode \
  --profile coding-plan_ap-southeast-1_accountwide \
  --model <coding-model-id>
```

For a plan profile:

- `--endpoint` is invalid;
- `--model` is optional;
- if `--model` is omitted, arkcli uses the selected or first model available
  from that profile;
- the selected profile must contain its stored default API key.

If the stored key is missing, recover with one of:

```bash
arkcli auth apikey
arkcli profile keys refresh
```

A one-off API key override does not replace the stored profile key used by
Helper plan configuration.

## Platform Profiles

For a Platform profile, the exact form is:

```bash
arkcli helper configure <harness> \
  --profile <platform-profile> \
  --endpoint <endpoint-id>
```

Platform rules:

- `--endpoint` is required;
- `--model` is invalid;
- the current identity must be an SSO sub-user;
- the Endpoint must be owned by that current sub-user;
- the Endpoint must be Running;
- the Endpoint must explicitly support text output.

Text-output multimodal Endpoints are allowed. Image-only, video-only, 3D,
audio-only, embedding-only, and unknown-output Endpoints are excluded.

When no eligible Endpoint exists, use the dedicated Endpoint commands:

```bash
arkcli infer endpoint create
arkcli infer endpoint start <endpoint-id>
```

BytePlus Helper does not invent a console fallback URL for this case.

## Client Matrix and Settings Paths

| Client | Harness ID | Settings path written or inspected | Reload or activation |
|---|---|---|---|
| Claude Code | `claude-code` | `$HOME/.claude/settings.json` | Restart or reload Claude Code. |
| Codex | `codex` | `$HOME/.codex/<profile>.config.toml` by default, or `$HOME/.codex/config.toml` for global scope | Run `codex --profile <profile>` for profile scope. Restart an existing session. |
| OpenCode | `opencode` | `$OPENCODE_CONFIG` or `$HOME/.config/opencode/opencode.json` | Restart or reload OpenCode. |
| OpenClaw | `openclaw` | `$OPENCLAW_CONFIG_PATH` or `$HOME/.openclaw/openclaw.json` | Restart or reload OpenClaw. |
| Hermes | `hermes` | `$HOME/.hermes/config.yaml` and `$HOME/.hermes/.env` | Restart or reload Hermes. |

`arkcli helper list` reports the Codex catalog path
`$HOME/.codex/config.toml`. This is an inspection hint, not the default write
scope. `configure codex` defaults to the named profile file
`$HOME/.codex/arkcli.config.toml`.

Trae is not supported by the BytePlus Helper command surface because its
current integration cannot write model provider configuration.

## Codex Scope

Default profile scope:

```bash
arkcli helper configure codex \
  --profile <profile-name> \
  --model <model-id>
codex --profile arkcli
```

The default Codex profile name is `arkcli`, and the default target is:

```text
$HOME/.codex/arkcli.config.toml
```

Custom profile:

```bash
arkcli helper configure codex \
  --profile <profile-name> \
  --model <model-id> \
  --codex-profile byteplus
codex --profile byteplus
```

Codex profile names may contain only letters, digits, underscores, and
hyphens.

Global scope:

```bash
arkcli helper configure codex \
  --profile <profile-name> \
  --model <model-id> \
  --codex-config-scope global
```

Global scope writes `$HOME/.codex/config.toml`. Use it only when the user
explicitly asks to change configuration shared by all Codex profiles.

## Reset

```bash
arkcli helper reset <harness>
```

For Codex, reset the same scope and named profile used during configuration:

```bash
arkcli helper reset codex \
  --codex-config-scope profile \
  --codex-profile byteplus
```

Reset removes only arkcli-managed provider/model entries and preserves
unrelated user configuration. It still writes immediately, and the Helper
domain does not register `--dry-run`. Show the exact target and obtain
explicit confirmation before running it.

## Errors and Recovery

| Symptom | Meaning | Recovery |
|---|---|---|
| Parent command cannot prompt | No interactive TTY is available. | Use `arkcli helper configure <harness> ...`. |
| Login or session error | The selected operation needs a valid BytePlus identity. | Run `arkcli auth login`, then retry. |
| Unsupported profile type | Helper accepts only Platform, Coding Plan, and Coding Plan Team profiles. | Inspect `arkcli profile show` and switch with `arkcli profile use <name>`. |
| Missing stored plan API key | The selected plan profile cannot provide credentials to the client. | Run `arkcli auth apikey` or `arkcli profile keys refresh`. |
| No user-owned Endpoint | No eligible Endpoint belongs to the current sub-user. | Run `arkcli infer endpoint create`. |
| Endpoint is not Running | The selected Endpoint cannot serve requests. | Run `arkcli infer endpoint start <endpoint-id>`. |
| Endpoint is filtered out | It is not an eligible text-output Endpoint. | Select or create a Running text-output Endpoint. |
| Unsupported harness | The client cannot receive model/provider configuration through BytePlus Helper. | Re-run `arkcli helper list` and choose a listed harness ID. |
| MCP or database add-on requested | That capability is outside the BytePlus Helper surface. | State the boundary; do not use another tenant or product as fallback. |

## Completion Receipt

After a successful write, report:

- the harness;
- the selected profile type and profile name;
- the model ID or Endpoint ID;
- the settings path or paths;
- the Codex scope and profile name, if applicable;
- the reload or `codex --profile <name>` activation step;
- whether the client was already installed or was installed by the interactive
  wizard.

Never print full API keys, tokens, or other credentials.
