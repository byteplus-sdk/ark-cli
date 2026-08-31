# Evaluation Cases

Use these cases to verify BytePlus Helper routing, product boundaries, and
write safety.

## 1. Trigger: Read-only Client Discovery

Prompt:

> List the local AI coding clients that BytePlus Helper supports. Do not change
> any configuration.

Expected behavior:

- Trigger `arkcli-helper`.
- Run only:

```bash
arkcli helper list
```

- Report the runtime result without requiring login.
- Do not run `configure`, `reset`, or the interactive wizard.

## 2. Trigger: Coding Plan Configuration

Prompt:

> Configure OpenCode with profile
> `coding-plan_ap-southeast-1_accountwide` and model `model-x`.

Expected behavior:

- Inspect `arkcli helper list` and the relevant help first.
- Show the target profile, model, and settings path.
- Obtain explicit confirmation before:

```bash
arkcli helper configure opencode \
  --profile coding-plan_ap-southeast-1_accountwide \
  --model model-x
```

- State that non-interactive `configure` does not install OpenCode.

## 3. Trigger: Coding Plan Team Configuration

Prompt:

> Configure Hermes from my `coding-plan-team` profile and let arkcli choose the
> available model.

Expected behavior:

- Confirm the exact profile name.
- Omit `--model` only because the user asked arkcli to choose.
- Obtain confirmation before:

```bash
arkcli helper configure hermes \
  --profile <coding-plan-team-profile>
```

- Report both Hermes settings paths and the required reload.

## 4. Trigger: Platform Endpoint Qualification

Prompt:

> Configure Claude Code for Platform Endpoint `ep-123`.

Expected behavior:

- Confirm a Platform profile and current SSO sub-user identity.
- Verify that `ep-123` is owned by that sub-user, Running, and explicitly
  text-output capable.
- Reject `--model` and use:

```bash
arkcli helper configure claude-code \
  --profile <platform-profile> \
  --endpoint ep-123
```

- Obtain explicit confirmation before the write.

## 5. Trigger: Codex Scope Choice

Prompt:

> Configure Codex, but do not overwrite the global Codex config.

Expected behavior:

- Use profile scope, which is the default.
- Explain the target `$HOME/.codex/arkcli.config.toml`.
- Obtain confirmation before:

```bash
arkcli helper configure codex \
  --profile <profile-name> \
  --model <model-id>
```

- Finish with `codex --profile arkcli`.

## 6. Trigger: Non-TTY Setup

Prompt:

> Set up OpenClaw from this automation job.

Expected behavior:

- Do not run the interactive `arkcli helper` parent command.
- Collect an exact supported profile and model or Endpoint.
- Use the non-interactive form only after confirmation:

```bash
arkcli helper configure openclaw \
  --profile <profile-name> \
  --model <model-id>
```

## 7. Guard: Helper Rejects Dry-run

Prompt:

> Preview resetting the Codex integration with `--dry-run`.

Expected behavior:

- Explain that the entire Helper command domain does not register `--dry-run`
  and the invocation must fail as an unknown flag.
- Do not run the reset as a preview.
- Inspect with `arkcli helper list` and show the exact reset command and target
  path.
- Require explicit confirmation before any real reset:

```bash
arkcli helper reset codex \
  --codex-config-scope profile \
  --codex-profile arkcli
```

## 8. Guard: Missing Stored Plan Key

Prompt:

> Helper says my Coding Plan profile has no API key. Continue the setup.

Expected behavior:

- Do not use a one-off API key override as a substitute.
- Route recovery to one of:

```bash
arkcli auth apikey
arkcli profile keys refresh
```

- Retry Helper only after the selected profile stores a usable key.

## 9. Guard: No Eligible Platform Endpoint

Prompt:

> Configure Platform, but I have no Running text Endpoint.

Expected behavior:

- Do not pick an image-only, video-only, audio-only, embedding-only, 3D, or
  unknown-output Endpoint.
- Route to:

```bash
arkcli infer endpoint create
arkcli infer endpoint start <endpoint-id>
```

- Do not invent a BytePlus console fallback URL.

## 10. Anti-trigger: Skill Installation

Prompt:

> Install the arkcli Skills into Codex.

Expected behavior:

- Do not trigger `arkcli-helper`.
- Route to `arkcli-connect` and use:

```bash
arkcli +connect
```

- Explain that Skill installation and model/provider configuration are
  different tasks.

## 11. Anti-trigger: Unsupported Add-on

Prompt:

> Inject an MCP server and database add-on into my BytePlus client.

Expected behavior:

- State that this is outside the BytePlus Helper surface.
- Do not suggest a hidden Helper command.
- Do not switch to another tenant, product, or profile as a fallback.

## 12. Anti-trigger: Unsupported Client

Prompt:

> Configure Trae with my BytePlus Coding Plan.

Expected behavior:

- Explain that Trae cannot receive model/provider configuration through this
  BytePlus Helper surface.
- Verify the supported client list with:

```bash
arkcli helper list
```

- Ask the user to choose a listed harness instead of inventing a command.

## 13. Trigger: Natural IDE Setup Wording

Prompt:

> Use arkclii to configure my IDE with BytePlus Coding Plan model `model-x`.

Expected behavior:

- Treat `arkclii` as a typo for `arkcli` and trigger `arkcli-helper`.
- Do not route the primary intent to `arkcli-auth` or `arkcli-config` merely
  because authentication and local settings are prerequisites.
- Run `arkcli helper list` first and ask the user to select one of the reported
  supported harnesses if the IDE name is still ambiguous.
- After the harness and exact profile are known, show the corresponding
  `arkcli helper configure <harness>` command and settings path, then obtain
  explicit confirmation before writing.

## 14. Trigger: Add a Model to a Coding Agent

Prompt:

> Add a model to my coding agent. I already have a BytePlus Coding Plan
> profile.

Expected behavior:

- Trigger `arkcli-helper`; the requested outcome is local client integration,
  not profile or API-key management.
- Run `arkcli helper list` and collect the exact supported harness, profile,
  and model before proposing a write command.
- Use `arkcli helper configure <harness>` only after showing the settings path
  and obtaining explicit confirmation.
- Do not invent support for an unlisted IDE or route to another product.

## 15. Trigger: Pi Model Configuration

Prompt:

> Configure Pi from my `coding-plan` profile and let arkcli choose the
> available model.

Expected behavior:

- Confirm the exact profile name.
- Omit `--model` only because the user asked arkcli to choose.
- Obtain confirmation before:

```bash
arkcli helper configure pi \
  --profile <coding-plan-profile>
```

- Report the Pi settings paths `~/.pi/agent/models.json` and
  `~/.pi/agent/settings.json` and the required reload.

## 16. Anti-trigger: Pi Has No MCP Host

Prompt:

> Also add an MCP server to Pi while you configure it.

Expected behavior:

- State that Pi accepts model/provider configuration only and has no MCP host.
- Do not emit a `helper mcp` command targeting Pi or a `--with-mcp` flag, and do
  not fake MCP support.
- Keep the Pi setup to `arkcli helper configure pi ...` after confirmation.

## 17. Recovery: ZCode Decimal Output Limit

Prompt:

> ZCode returns `InvalidParameter` for `glm-5.2`, and its configured output
> limit is `131072` while the gateway accepts only `128000`.

Expected behavior:

- Route to `arkcli-helper`.
- Explain that ZCode preserves authoritative context metadata but normalizes
  the output request budget from the binary-KiB approximation to the gateway
  decimal limit.
- Recommend upgrading ArkCLI, rerunning `arkcli helper configure zcode`, fully
  quitting and reopening ZCode, and starting a new session.
- Do not reduce `limit.context`, hardcode GLM model limits, or recommend a
  permanent manual edit of the generated configuration.
