---
name: arkcli-deploy
version: 1.4.0
description: "arkcli +deploy: The unified preferred entry point for creating inference endpoints (Endpoint) —— **when users say “create/new/create an endpoint” or “deploy/launch/deploy a model,” as long as the intent is to create a new endpoint, always use this first, and do not use create in arkcli-infer-endpoint**. Use this when the user needs to deploy a model as an online inference endpoint. Note: Use arkcli-infer-endpoint only when performing full lifecycle management such as get/list/start/stop/update on an **existing** endpoint; this skill is only responsible for one-click creation (CRUD beyond creation is not included here). "
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli +deploy --help"
---

# arkcli +deploy

**Prerequisite:** First use Read to read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) to get shared authentication/configuration/write-operation guard rules.

**New flag `--set-default <modality>`**: After a real deployment succeeds, automatically set the new endpoint as the default resource for this modality (`text` / `image` / `video`) in the active profile when the user explicitly passes a modality; failures only warn on stderr and do not block the main deployment flow. For details, see [`../arkcli-shared/references/profile-defaults.md`](../arkcli-shared/references/profile-defaults.md).

**Write operation + billing**: `+deploy` creates an online inference endpoint. Its workflow depends on live discovery and **does not support `--dry-run`**. BytePlus enforces a two-stage runtime gate:

1. Run the exact command **without `--yes`**. The CLI performs only the read-only discovery needed to normalize the request, returns `status: requires_confirmation` with the final `model`, `endpoint_name`, `region`, `configuration`, and `billing_impact`, and does not activate a model or create an Endpoint.
2. Show those exact returned values to the end user and stop. Do not treat confirmation given before this disclosure as approval of the normalized plan, and do not add `--yes` in the same turn.
3. Only after a **fresh explicit confirmation** of the disclosed values may you rerun the **same command with `--yes`**. This second invocation may create the billable Endpoint.

**Model activation is a separate billable write action and must never be automatic in non-interactive environments**: If the target base model has not been activated, the confirmed `+deploy --yes` invocation reaches the separate model-activation gate. **In non-TTY environments such as agent / CI / pipelines, activation is hard-rejected even when `--yes` is present**: `--yes` confirms the previously disclosed Endpoint creation plan, but it does not authorize model activation. When this is hit, the CLI returns `model_activation_required` plus a console link. You **must end this turn and hand the activation billing action together with the link back to a human**, who must confirm in an interactive terminal or activate it in the web console. It is strictly forbidden to add `echo Y` or set `ARKCLI_ALLOW_HEADLESS_ACTIVATION` yourself. `ARKCLI_ALLOW_HEADLESS_ACTIVATION=1` is reserved only for genuine unattended automation, not something an agent should set. If the user's request literally is "activate model without prompts", that is a **models activate** intent; route to `arkcli models activate`. Do not translate it into `+deploy --yes`, and do not synthesize a deployment name from the model name.

**Hard boundary for voice models**: For TTS / ASR / dubbing / reading aloud / podcasts / voice design / real-time voice interaction, **do not execute `+deploy`**. These models currently do not support endpoint creation in arkcli, and this cannot be bypassed by activating the model.

## Subcommand enumeration (only this one)

| Invocation |Description|
|------|------|
| `arkcli +deploy --name <ep-name> --model <model-id> [...]` | First stage: returns the normalized confirmation plan; creates nothing. |
| `arkcli +deploy --name <ep-name> --model <model-id> [...] --yes` | Second stage: after the fresh confirmation, creates the disclosed Endpoint. |

> ⚠️ **There is no** `arkcli deploy ...` / `arkcli endpoint create` / `arkcli +deploy create` or other subcommands. The entire capability is one `+deploy` command plus flags.

## Anti-hallucination checklist

- `--name` and `--model` are required.
- Model versions, prices, and quotas must come from structured results returned by read-only commands in the current run; **you must not invent model versions or prices**. If the query fails or authentication is missing, mark the value as unverified instead of substituting a "typical" price or guessed value.
- **After a read-only query fails, do not fill dependent fields**: keep the affected authentication, model, pricing, or resource facts unverified and name the failed source. Do not continue by asserting amounts, idle-charge policy, an exact model version, or that no existing Endpoint exists.
- To check existing Endpoints, run `arkcli resources list --format json` and filter the returned fields; **you must not add undocumented `--filter`** or any other flag absent from `resources list --help`.
- For JSON-type flags (`--rate-limit` / `--moderation` / `--intelligent-router` / `--tags`, etc.), parameter names must always be **PascalCase**: `Rpm`, `Tpm`, `Strategy`, `Mode`, not `rpm`/`tpm`.
- When the model has not been activated, activation in `+deploy` is **hard-rejected under non-TTY, and `--yes` does not allow it**; **do not add `--yes` / `echo Y` / set `ARKCLI_ALLOW_HEADLESS_ACTIVATION` yourself**. You must hand activation (billing) back to a human to handle in the terminal / console.
- For Endpoint creation, the first command must be run without `--yes`. Surface the CLI's `requires_confirmation` payload and wait for a fresh explicit confirmation before using the same command with `--yes`; never execute both stages in one agent turn.
- Deployment of voice models (TTS / ASR / podcasts / voice design / real-time voice interaction) is not supported; when such a model is hit, do not execute the `+deploy` command, and there is no need to execute any other command.

## Routing decision

- The user already has a model ID and wants to deploy formally -> run `arkcli +deploy --name <ep> --model <id>` without `--yes`, surface its normalized confirmation payload, wait for a fresh explicit confirmation, and only then rerun the same command with `--yes`.
- The user's verb is only "activate / open service / enable base|context-cache" (without endpoint/name) → **do not** run `arkcli +deploy`, route to `arkcli models activate`.
- The user wants to deploy / access a voice model, or the model name looks like `*-tts-*` / `*-asr-*` / `seedasr-*` → directly state that “endpoint creation is not supported.”
- When the user provides a custom model ID (`cm-xxxxx`), before real creation, it first checks whether there is an existing endpoint that references this custom model and has status `Running`; if yes, it directly reuses it and outputs the existing `endpoint-id`, without creating a second billable resource. This live reuse decision is one reason `+deploy` cannot provide a reliable offline Client Preview.
- The user urgently asks to "create it immediately" -> **do not skip either stage**. Run only the no-`--yes` disclosure stage, then stop for a fresh explicit confirmation.
- If an `Id` has already been obtained through `arkcli infer endpoint create` → **do not** create a second one with `+deploy`; route to `arkcli-infer-endpoint`.

## Anti-triggers (route elsewhere, with full commands to avoid downstream hallucinations)

| User intent | Route to | Full example command |
|---------|--------|------------|
| Only want to try the model output / one-off generation | `arkcli-chat` / `arkcli-gen` | `arkcli +chat --model <id> '...'` or `arkcli +gen --model <id> '...'` |
| Model ID not decided | `arkcli-models` | `arkcli models search <keyword>` or `arkcli models list` |
| 401 / authentication failed | `arkcli-auth` | `arkcli auth status`; if needed, `arkcli auth login` |
| profile / region / project does not match expectations | `arkcli-config` | `arkcli profile show --format json` (the old `arkcli config show` is deprecated) |
| Scripting / CI / need fine-grained control over every parameter and to skip guardrails | `arkcli-infer-endpoint` | `arkcli infer endpoint create --model <id> --name <ep>` |

## Typical workflow

1. **From model selection to production access**: `arkcli auth status` -> `arkcli models search/get` -> `arkcli +deploy ...` without `--yes` -> show the returned final model/name/region/configuration/billing impact -> wait for a fresh explicit confirmation -> run the same command with `--yes`. (Custom models reuse an existing Running endpoint if one exists.)
2. **From trial to production access**: Verify the output with `arkcli +chat` / `arkcli +gen`, then follow the same two-stage disclosure and confirmation flow.

For detailed flags, JSON parameter examples, and error codes, see [`references/arkcli-deploy.md`](references/arkcli-deploy.md).

## References

- [arkcli-chat](../arkcli-chat/SKILL.md) -- Quick chat trial; does not create an endpoint.
- [arkcli-gen](../arkcli-gen/SKILL.md) -- One-step image/video generation.
- [arkcli-models](../arkcli-models/SKILL.md) -- Confirm the model ID before deployment.
- [arkcli-shared](../arkcli-shared/SKILL.md) -- Authentication and global parameters.
