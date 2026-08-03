---
name: arkcli-models
version: 1.0.0
description: "arkcli model query capability: activate, list, search, and get details for BytePlus ModelArk public foundation models. Prefer product commands `arkcli models ...` over direct Raw API calls. Note: use arkcli-custommodel to query/manage custom models uploaded or fine-tuned (`cm-xxx`) under the account; this skill covers only the public foundation model catalog. Voice/TTS/ASR/podcast/voice-design/real-time voice interaction models support only marketplace search and selection guidance; do not route them to +chat/+gen/+deploy/usage/pricing/onboard/auth apikey."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli models --help"
---

# arkcli models

**CRITICAL — Before starting, you MUST use the Read tool to read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md), which contains authentication gates, configuration troubleshooting, and command selection order.**
**CRITICAL — Before executing any `models` command, you MUST use the Read tool to read its corresponding reference document. Do not invoke commands blindly.**

## Usage principles

- Prefer `arkcli models ...` for model-related requests.
- Although these are standard CLI commands, their implementation entry point still comes from `shortcuts/models/`.
- Fall back to [`../arkcli-api-explorer/SKILL.md`](../arkcli-api-explorer/SKILL.md) only when product commands cannot cover the request.
- This skill is not the default fallback entry point. Enter it only when the user explicitly asks for model queries, model asset inventory, model details, or model selection for an upstream command.
- **Voice model boundary**: Voice models such as TTS / ASR / podcast / voice design / real-time voice interaction support only marketplace search and selection guidance in arkcli; there are currently no supported voice models. Finding one with `models search` does not mean it can be used with `+deploy`, `usage`, or `pricing`. Unless the user separately asks for official documentation, do not proactively provide non-arkcli integration steps or links for the console / OpenAPI / SDK.

## Applicable scenarios

- The user wants to search, filter, or compare BytePlus ModelArk models.
- The user wants to view model details, versions, context, modalities, or lifecycle states.
- The user wants to count or list "my models", "custom models", or "recently created models".
- The user wants to activate model / open model service / enable base | context-cache for a model.
- Upstream `+chat` / `+gen` / `+deploy` needs to determine an available model name first.
- The user asks whether voice models exist, which voice models are available, or what TTS/ASR models are called in the marketplace: answer only the marketplace search facts and the boundary that arkcli currently does not support downstream scenario capabilities.

## Anti-trigger signals

- The user only wants to chat directly, generate images/videos, or deploy an Endpoint: route to the corresponding skill first and return here only if the model name is missing.
- The user explicitly requests a raw Action, OpenAPI listing, or low-level params: route to `arkcli-api-explorer`.
- The user is troubleshooting authentication, profile, region, or base URL: route to `arkcli-auth` or `arkcli-config`.
- Do not treat `arkcli models` as a default entry point; do not query models first for non-model questions.

## Business positioning

- This skill is a dependency of other business workflows, not a destination skill.
- It normally serves three types of upstream goals:
  - Find the correct model for `+chat` / `+gen`.
  - Confirm a deployable model for `+deploy`.
  - Confirm model details and versions for business troubleshooting.
- Unless the user is explicitly performing a model query, return to the original task after finding the model.
- **Exception**: A voice model query is itself the destination. After finding marketplace voice models such as podcast, voice design, or real-time voice interaction, stop after explaining "discoverable but arkcli does not support invocation/deployment/usage/pricing". Do not continue to `+deploy` / `usage` / `pricing` / `onboard`, and do not proactively add non-arkcli integration paths.

## Quick decisions

### Step 0 (hard gate): read the scenario table before selecting a command

**This is the first action for any "find a model by intent" request, before choosing `search` / `list` / `resources`. You MUST use Read to open [`references/arkcli-models-scenario-table.md`](references/arkcli-models-scenario-table.md), determine whether the intent matches a scenario label, and only then decide which command to run.** This manually curated `scenario label → recommended model` table is the **highest-priority signal for intent ranking and overrides all naming heuristics / modality / capability filtering below**.

**Counterexamples that are easily intercepted (must read): the following intents may "sound like hard metrics", but are actually scenario labels. Check the scenario table first instead of instinctively dropping to the `--capability` / `--modality` fallback path:**

| User says | ❌ Do not use directly | ✅ Scenario label matched in Step 0 → recommended |
|---|---|---|
| "Complex reasoning / multi-step / strongest-quality model" | `search --capability thinking` | Complex reasoning / Agent tasks → `dola-seed-2-1-turbo` |
| "Which model should I use for image generation?" | Stop after `resources list --modality image` | Image generation → `dola-seedream-5-0-pro` |
| "Video / role-play / field extraction" | `search --modality ...` | Video generation / role-play / information extraction → consult the table |

After a match, follow the JOIN protocol in the table: take the recommended model family stem → run `arkcli models search <family stem>` to verify facts against the left table → rank the recommendation first + add 2–3 alternatives. **Versions/full IDs in the table are only starting points; use the real-time search response for matches** (actual testing shows that table IDs can become stale).

**Skip the scenario table and fall back to `search` + naming heuristics below only in these two cases:**
- The user provides an **explicit numeric/capability hard constraint** ("200K context", "must support functioncall"), and the scenario table has no corresponding label.
- The user **names a third-party / open-source / historical model** (qwen, glm, Seedream 4.5, and others); do not forcibly replace it.

> Distinguish carefully: "complex reasoning" is a scenario label (table → pro) and **does not mean** the user supplied the `thinking` hard metric. Only the latter uses the fallback path.

**Layer boundary (important; do not mix layers): the scenario table is the "model marketplace selection layer". It answers "which model should be selected in the marketplace for this intent", and its data source is always `models search` (catalog).** It provides catalog-recommended model names (for example, image generation → `dola-seedream-5-0-pro`).

- "Whether the current profile can invoke it and the exact callable ID" belongs to the **downstream availability layer**, handled by `resources list` → `models get` → `+gen` / `+chat` (arkcli-gen is implemented), and is **outside this table's responsibility**.
- Therefore, for a **pure selection question** ("find/choose a model") → use only `models search`; **do not** query `resources list` and turn "selection" into "available-resource lookup". Descend to resources list to resolve the current profile's actual ID only when the user **truly wants to generate / invoke**.

This applies only to the **intent-ranking path**. Enumeration / inventory / statistics is another path: use `list` without scenario-table reranking.

### Command selection

- The user knows only a fuzzy model name / wants to find by intent ("strongest video model", "200K-context LLM", "model supporting thinking"): use `search`, which provides fuzzy keyword matching + structured modality/context/capability filtering.
- To enumerate by modality / pagination parameters, or perform exact-name matching: use `list`.
- The user asks "my models", "custom models", "how many were recently created", "list them", or "count them": this is model asset inventory, not candidate discovery. First read [`references/arkcli-models-list.md`](references/arkcli-models-list.md), use `arkcli models list --page-all`, and perform client-side filtering. Do not jump to Raw API Explorer.
- A specific model ID is already known: use `get`.
- The purpose is only to find a model for `+chat` / `+gen` / `+deploy`: return to the original task immediately after finding it.
- The user **proactively** wants to activate a foundation model ("activate dola-seed-2-1-turbo first", "preview the activation request first"): use `activate`, first reading [`references/arkcli-models-activate.md`](references/arkcli-models-activate.md). If the user only wants to deploy / create an endpoint, let deploy / infer-create trigger implicit activation and do not activate it separately first.

## Agent quick execution order

1. Confirm whether the user is logged in; if uncertain, first inspect `arkcli auth status`.
2. When the user's description includes intent (modality / numeric capacity / capability), use `arkcli models search` + corresponding flags (`--modality`, `--min-context-window`, `--capability`, and others).
3. When the user knows only a fuzzy name, still use `arkcli models search <keyword>` (returns all matches by default, without pagination).
4. To fully enumerate by modality or calculate pagination statistics, use `arkcli models list`.
5. When the user asks for "custom models/my models created in the past N days", use `arkcli models list --page-all --sort-by CreateTime --sort-order Desc --format json`, then locally filter by `create_time`, `model_type` / `customization_type` / `source_type`, and other fields. Do not explore `arkcli api --list` merely because there is no time-filter flag.
6. When a specific model ID is known and details are needed, use `arkcli models get`.
7. When the user explicitly wants model activation (proactive activation, not immediate deployment), use `arkcli models activate <name> [--sub-services ...]`; add `--yes` in CI and use `--dry-run` only for local request preview, never as server validation.
8. After finding / activating the model, return to the upstream workflow that initiated it: `+chat` / `+gen` / `+deploy`.

## Typical business workflows

### 0. Complete the full model ID for data-plane commands (required prerequisite)

The `--model` argument for `+chat` / `+gen` must use the full `<name>-<primary_version>` form (or Endpoint ID `ep-xxx`). **Passing only the family name triggers `InvalidEndpointOrModel.NotFound` 404.**

The **`primary_version` format is not fixed**. **Do not use a regex to decide whether it "looks like a full ID".** Formats actually observed (about half of ~152 models do not use six digits):

| Format | Example full ID | Common family |
|------|--------------|----------|
| `YYMMDD` (six-digit date) | `dola-seed-2-1-turbo-260628`, `seed-2-0-lite-260428` | dola family |
| Qualifier prefix + date | `seed-2-0-code-preview-260328` | Preview / variant |

**Two acquisition paths, in decreasing order of efficiency:**

- **Path A (efficient; reuse existing results)**: After running `arkcli models search <keyword>` or `arkcli models list`, each returned JSON item includes `primary_version`. The Agent directly reads it and concatenates `<name>-<primary_version>` without an additional call. **`search` also returns structured fields such as `context_window` / `input_modalities` / `output_modalities` / `capabilities` (from ArkModels enrich). Downstream logic can determine suitability directly and avoid an extra `get` call.**
- **Path B (standalone query)**: When only a bare name is known, `arkcli models get <name> --transform 'primary_version'` returns the version number directly.

**Note**: `--transform` output includes JSON double quotes (for example, `"260328"`). Before shell concatenation, remove them with `tr -d '"'`:

```bash
VER=$(arkcli models get dola-seed-2-1-turbo --transform 'primary_version' | tr -d '"')
# VER=260628  -> full ID: dola-seed-2-1-turbo-260628
# VER="" (model with an empty string) -> use the family name directly: dola-seed-2-1-turbo
if [ -n "$VER" ]; then
  MODEL="dola-seed-2-1-turbo-$VER"
else
  MODEL="dola-seed-2-1-turbo"
fi
arkcli +chat --model "$MODEL" "hello."
```

### 1. Find a model for trial use

`models search [--modality / --capability ...]` → `+chat` / `+gen`

### 2. Find a model for production integration

`models search [--min-context-window / --capability ...]` → `+deploy`

### 3. Filter by hard metrics

```bash
# Find the newest and strongest text model with 200K+ context and thinking capability
arkcli models search --modality text --min-context-window 200000 --capability thinking --strict-filter

# Find video generation models
arkcli models search --modality video --strict-filter

# Find multimodal LLMs
arkcli models search --multimodal --output-modality text --strict-filter
```

Adding `--strict-filter` is strongly recommended: the default behavior is "retain missing data" (to avoid false negatives); strict returns only models that satisfy the conditions with 100% certainty.

## Common fallbacks

- Authentication failure: route to [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md).
- Profile / region looks wrong: route to [`../arkcli-config/SKILL.md`](../arkcli-config/SKILL.md).
- If the business goal is only to find a model name for `+chat`, `+gen`, or `+deploy`, prioritize "find the model, then return to the original task"; do not stop at models itself.

## Command overview

| Command | Description |
|------|------|
| `arkcli models search [keyword] [filters]` | **Agent's first choice**: full recall + ArkModels enrichment + structured modality/context/capability filtering + reranking; returned fields include `context_window` / `input_modalities` / `output_modalities` / `capabilities` |
| `arkcli models list` | Full enumeration by modality, pagination statistics, and model asset inventory; lightweight, without enrichment |
| `arkcli models get <id> [version]` | Complete details for one model (aggregates multiple APIs; heaviest and most complete) |

## Tier naming heuristics (fallback only when the scenario table has no match)

**Priority mnemonic: scenario-table match (★★★) > naming heuristics (★★, this section) > update_time.** First use the scenario table in "Quick decisions Step 0". Use this section's naming heuristics only when the scenario table does not cover the intent or the user names a third-party/open-source/historical model.

After `search` and filters, multiple candidates often remain. Tier signals in model names can help the Agent perform the final ranking step. **This is a heuristic, not a hard rule**: hard metrics (context_window / capabilities / modality) always take precedence over naming.

### Generation (major version): highest priority

```
2-x > 1-8 > 1-6 > 1-5 > ...
```

A generation jump usually matters **more than tier**. Example: `dola-seed-2-1-turbo` is stronger than `seed-2-0-pro` for most tasks because the base-model generation gap is larger than the size-tier gap.

### Tier (size tier within the same generation)

```
pro  ≥  no suffix  >  lite  >  flash  >  mini  >  nano
```

- `pro`: flagship, largest size / strongest capability
- No suffix (such as `dola-seed-2-1-turbo`): mainstream tier, normally ≈ pro
- `lite`: cost/performance balance
- `flash`: speed optimized
- `mini` / `nano`: low latency, high concurrency, lowest cost

### Specialty suffixes (not compared as tiers; select by use)

```
-code        → programming optimized (seed-2-0-code-preview, and others)
-thinking    → enhanced thinking capability (longer reasoning)
-vision      → enhanced vision understanding
-character   → role-play / long narration
-translation → translation-specific
```

A specialty model is **normally stronger than the same-generation general pro in its domain**, but does not transfer across tasks.

### Notes on comparing `primary_version`

- **Within the same family**: newer dates are better (`260215 > 251228 > 251015`).
- **Across different families**: values **cannot be compared numerically**. `glm-5-2-260617` and `seed-2-0-pro-260328` both use six digits, but their numeric order says nothing about which is stronger.
- To compare "which is newer", always use the `update_time` field (ISO 8601, lexicographically comparable); do not parse `primary_version`.

### Decision mnemonic

```
filter (hard metrics; must be satisfied)
   ↓
generation (2-x > 1-x; cross-generation gap usually > same-generation tier gap)
   ↓
tier (pro > no suffix > lite > flash > mini > nano)
   ↓
update_time (when generation and tier match, newer first)
```

### Sorting is already implemented in `search`

The Agent does not need to perform client-side reordering. `arkcli models search` uses two-stage sorting: "group by family first, then sort within each group":

```
1. bucket             — keyword visibility (name match > description match > fallback > hidden)
   ↓
2. across families:   sort by the family's "representative score" (max ctx among family members → max update_time → family name)
   ↓                  → all members of the same family remain together
3. within a family:   gen DESC → tier DESC → ctx DESC → update_time DESC → name ASC
```

**Key design points**:
- **Family grouping precedes detailed sorting**. The dola-seed family (max ctx=260628) as a whole ranks before all families with ctx=null. This avoids non-transitive oddities such as "1-6-flash in the same family ranking above 1-8".
- **Within the same family, generation precedes ctx**. A new version floats above an older version with existing ctx data even if ArkModels metadata has not yet been completed (`context_window=null`). Example: `glm-5-2` (gen=502, ctx=null) ranks before `glm-4-7` (gen=407, ctx=251222).
- **Generation precedes tier**. Within the same family, `dola-seed-2-1-turbo` (gen=201, tier=50) ranks before `seed-2-0-lite` (gen=200, tier=80), because cross-generation gaps are normally greater than same-generation tier gaps.
- **Do not compare generation/tier across families**. pro/lite/mini are naming conventions within a family and are not comparable across families. Family order is determined by the representative score (max ctx + max time).

Legacy models tagged with any of `experience hiding` / `inference hiding` / `Square Hide` automatically sink to the bottom (they are not filtered from list/search but always appear last).

> **Note**: `experience hiding` / `inference hiding` / `Square Hide` are legacy hidden-page tags in the BytePlus ModelArk platform's `customized_tags`. They are unrelated to this CLI command; arkcli reads them only for sorting.

## Model lifecycle: Shutdown / Retiring / Published

ArkModels assigns each model a `lifecycle_status` with three values, handled differently by Search:

| status | Meaning | Default search behavior |
|--------|------|----------------|
| `Published` | Normal service | ✓ Display |
| `Retiring` | Being retired (still callable; not recommended for new integration) | ✓ Display; the Agent should verbally tell the user "X is being retired; use Y instead" |
| `Shutdown` | Offline (calls will fail) | **❌ Filtered by default** (returns only with `--include-deprecated`) |

In addition, models whose `display_name` contains `abandoned` / `offline` / `not in service` / `deprecated` are handled as Shutdown (fallback because some legacy models lack ArkModels metadata and rely on manual labels).

**Agent behavior contract**:
- No client-side logic is needed to "filter Shutdown"; search already does it by default.
- When a model with `lifecycle_status="Retiring"` appears, **proactively tell the user it is being retired** and try to find a newer version in the same family (use `search` + the correct keyword).

## ❌ Wrong patterns (Agents must read)

The following behaviors are wrong. Choosing the wrong command prevents the Agent from obtaining enrichment data (context_window / modalities / capabilities), hidden-item sinking, and weighted reranking, leading to incorrect recommendations.

- ❌ **Do not use `list` to find a model**. Any "which model/find an X model" intent must use `search`. `list` does not perform fuzzy keyword matching or weighted reranking and does not return enrichment fields.
- ❌ **Do not use `list --modality video` to select a video generation model**. Use `arkcli models search --modality video --strict-filter`; it combines ArkModels `output_modalities` with a `task_types` fallback, recalls more accurately, and supports additional `--min-context-window` / `--capability` filters.
- ❌ **Do not use `list --modality text` to select an LLM**. Likewise, use `search --modality text --strict-filter`. `list --modality text` considers only one signal from foundation_model_tag.filter_domains and misses many models.
- ❌ **Do not run `list`, then `get`, to verify context_window and similar parameters**. `search` already returns `context_window` / `max_input_tokens` / `max_completion_tokens`, saving one `get` call.
- ❌ **Do not omit the keyword from `search` merely to "view the five most popular models"**. Omitting the keyword now returns **all 152 entries** sorted by UpdateTime descending; it is no longer a curated popular list. Use `--size 5` for a small result.
- ❌ **Do not recommend a model with `lifecycle_status="Retiring"` for new integration**. Although still callable, the vendor has marked it for retirement. Proactively warn the user and search for a newer version.
- ❌ **Do not perform client-side secondary filtering on `search` to compensate for list limitations**. Use `search`'s built-in `--modality` / `--min-context-window` / `--capability` flags directly.
- ❌ **Do not call `get` directly for one model without trying `search`**. `search <name>` returns all candidates + enrichment in one call. Use `get` only when pricing/rate limits/detailed capability descriptions are needed.
- ❌ **Do not explore Raw APIs with `arkcli api --list` merely because `models list` has no server-side time-filter flag**. First run `models list --page-all --format json`, then filter local JSON by `create_time`.
- ❌ **Do not tell the user "the CLI does not have this capability; use the console"**, unless you have confirmed that the current `arkcli models list --help` truly has no enumerable output and local JSON filtering also cannot complete the requested statistics.

The only valid uses of `list`:
- ✓ Perform **full enumeration/auditing** by `--modality` (not to "find the strongest").
- ✓ Obtain statistics such as `total_count`.
- ✓ Exact matching with `--name foo` (rarely needed by Agents because search also finds it).
- ✓ Inventory assets such as "mine/custom/recently created": retrieve all with `--page-all`, then filter client-side by fields and time window.

## References

- [arkcli-chat](../arkcli-chat/SKILL.md) -- Enter the chat / inference workflow after finding a model
- [arkcli-gen](../arkcli-gen/SKILL.md) -- Enter the generation workflow after finding an image/video model
- [arkcli-deploy](../arkcli-deploy/SKILL.md) -- Enter the Endpoint deployment workflow after finding a model
- [`references/arkcli-models-list.md`](references/arkcli-models-list.md)
- [`references/arkcli-models-get.md`](references/arkcli-models-get.md)
- [`references/arkcli-models-search.md`](references/arkcli-models-search.md)
- [`references/arkcli-models-scenario-table.md`](references/arkcli-models-scenario-table.md) -- Scenario recommendation table (highest priority for intent ranking) + JOIN verification protocol
- [`references/arkcli-models-activate.md`](references/arkcli-models-activate.md)
