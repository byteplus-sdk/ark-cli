# models search

> **Prerequisite:** Read [`../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

**Agent-first** model search: recall = full catalog; enrichment = ArkModels (context window / modalities / capabilities); filtering = structured conditions; ranking = four buckets + context_window weighting + recency. There is **no concept of pagination**, and all matches are returned by default.

## Commands

```bash
# Basic
arkcli models search                       # All entries sorted by UpdateTime descending
arkcli models search doubao                # Keyword (or use --keyword doubao)
arkcli models search doubao --size 5       # Explicitly truncate to five entries

# Modality filtering (required for D3)
arkcli models search --modality video      # Video generation models
arkcli models search --modality text       # Text output models
arkcli models search --modality image      # Image generation models
arkcli models search --input-modality text,image --output-modality text  # VQA category
arkcli models search --multimodal          # Models whose input supports multiple modalities

# Numeric capacity filtering
arkcli models search --min-context-window 200000
arkcli models search --max-context-window 1000000
arkcli models search --min-input-tokens 100000
arkcli models search --min-output-tokens 8192

# Capability filtering (repeatable, AND relationship)
arkcli models search --capability thinking
arkcli models search --capability mcp --capability functioncall

# Composite query: find the strongest thinking LLM with 200K+ context
arkcli models search --modality text --min-context-window 200000 --capability thinking --strict-filter

# Cache control
arkcli models search --refresh-cache       # Force one synchronous refresh of ArkModels metadata
```

## Parameters

| Parameter | Type | Description |
|------|------|------|
| `[keyword]` / `--keyword` | string | Keyword. Matches any name / display_name / short_name / description / introduction field (case-insensitive) |
| `--size` | int | **Omitted = no truncation** (the Agent receives all entries by default); explicitly pass N to truncate to N |
| `--modality` | string | Coarse-grained alias: `text` / `image` / `video` / `audio` (output modality shorthand) |
| `--input-modality` | string | Explicit input-modality filter, comma-separated (such as `text,image,video`) |
| `--output-modality` | string | Explicit output-modality filter |
| `--multimodal` | bool | Return only models with at least two input modalities |
| `--min-context-window` | int | Minimum context_window (tokens) |
| `--max-context-window` | int | Maximum context_window |
| `--min-input-tokens` | int | Minimum max_input_tokens |
| `--min-output-tokens` | int | Minimum max_completion_tokens |
| `--capability` | string (repeatable) | Required capabilities; multiple values use AND: `thinking` / `mcp` / `functioncall` / `web_browsing` / `knowledge_base` / `caching` / `response_format` / `reasoning_effort` |
| `--strict-filter` | bool | Directly exclude models without enrichment data (retained by default to avoid false negatives) |
| `--include-deprecated` | bool | Include models with `lifecycle_status=Shutdown` or `display_name` containing "deprecated/offline" (**filtered by default**, retaining only callable models) |
| `--refresh-cache` | bool | Synchronously refresh the ArkModels metadata cache (stale-while-revalidate asynchronous refresh by default) |

## Workflow

```
search [keyword] [filters]
   │
   1️⃣  ListFoundationModels(all, sort=UpdateTime DESC)
   │   ↓ 152 candidates
   2️⃣  EnrichWithMetadata
   │   ├─ read cache: ~/.arkcli-bp/cache/<profile>/<region>/<project>/arkmodels-meta.json
   │   ├─ hit  → return immediately + fork detached subprocess for asynchronous refresh
   │   └─ miss → synchronously call ArkModels(IsPrimaryVersionOnly:true) → write cache
   │       (without SSO, enrichment is silently skipped and filtering automatically becomes a no-op)
   │   ↓ add context_window / input_modalities / output_modalities / capabilities to each item
   3️⃣  Modality fallback: derive from task_types when output_modalities is empty
   │   (TextToVideo → out=[video], VisualQA → in=[text,image] out=[text], ...)
   4️⃣  Keyword filtering (case-insensitive substring across name+display+short+description+intro)
   5️⃣  Structured filtering (modality / context / tokens / capability, AND)
   6️⃣  Four-bucket reranking + (context_window desc, update_time desc, name asc) tie-break
   7️⃣  --size N truncation (no truncation by default)
```

## Return fields

Each item:

```json
{
  "name": "seed-2-0-pro",
  "display_name": "Dola-Seed-2.0-pro",
  "primary_version": "260328",
  "update_time": "2026-03-28T...",
  "foundation_model_tag": { "filter_task_types": [...], "task_types": [...] },

  // —— enrichment (from ArkModels; null when SSO fails) ——
  "context_window": 262144,
  "max_input_tokens": 229376,
  "max_completion_tokens": 65536,
  "input_modalities": ["text", "image", "video"],
  "output_modalities": ["text"],
  "capabilities": {
    "thinking": true, "mcp": true, "functioncall": true,
    "web_browsing": true, "knowledge_base": true, "caching": true,
    "response_format": false, "reasoning_effort": true
  }
}
```

## Important behavior

### Handling missing data (StrictFilter)

- **Default `--strict-filter=false`**: with `--modality video`, models **without** modality data are also **retained** (the Agent sees candidates with low confidence).
- **`--strict-filter`**: missing data is treated as not satisfying the condition and is excluded directly (100% of the Agent's results are ground truth).

Modality queries are normally more accurate with `--strict-filter`; numeric queries (context window) are usually more reasonable with the default behavior (do not eliminate missing data).

### Recall no longer has a top-K limit

Unlike the old fuzzy API's default of nine entries, this command returns **all matches** by default. To limit results, pass `--size N`.

### Cache and SSO

- ArkModels is a Console BFF API and requires SSO credentials.
- Without SSO (AK/SK only) → enrichment is silently skipped, models are still returned, and filtering automatically becomes a no-op.
- Cache paths are separated by `(identity/account, profile, region, project)`. Persistent caching is disabled when no authoritative identity is available, preventing cross-account reuse.
- The default TTL is 5 minutes and can be adjusted with `ARKCLI_MODEL_CACHE_TTL`. Reads within the TTL use local cache only.
- After TTL expiry, stale-while-revalidate serves the current snapshot and starts a detached refresh (`arkcli models _refresh-cache`). Cold fills and refreshes are single-flighted across local processes for the same scope.
- `auth logout` and a fresh account switch remove all model caches for the current product.
- Every ArkModels Action passes through an account-scoped transport limiter with a 250ms minimum interval (about 4 QPS, leaving headroom below the backend's 5 QPS limit).

### Keyword matching scope

- name / display_name / short_name (match → reranking bucket 1)
- description / display_description / introduction (match → bucket 2)
- Substring matching, **case-insensitive**
- **No semantic/synonym expansion**

### Reranking details

| Bucket | Condition |
|---|------|
| 1 | Non-hidden + name matches keyword |
| 2 | Non-hidden + description matches keyword |
| 3 | Non-hidden + no keyword match (fallback) |
| 4 | Hidden (`customized_tags` contains `experience hiding`/`inference hiding`/`Square Hide`; these are legacy hidden-page tags from BytePlus ModelArk and are unrelated to this CLI command) |

Within a bucket: larger `context_window` first → newer `update_time` first → `name` lexicographically.

## Common errors

| Error | Cause | Handling |
|------|------|------|
| `--modality video` returns substantial noise | strict-filter=false by default, so models with missing data were not excluded | Add `--strict-filter` |
| Unexpected results after numeric filtering | Some ArkModels entries lack context_window | Accept the default retention (items with ctx=null may be inaccurate) |
| No thinking models | --capability is non-strict by default and results may be overwhelmed by noise | Add `--strict-filter` |
| All enrichment fields are null | Not logged in through SSO | Run `arkcli auth login` to re-establish BytePlus identity |
| Recall is clearly lower than expected | Cache is old, or ArkModels may be temporarily unavailable | Add `--refresh-cache` to force a synchronous refresh |

## Division of responsibility with `models list` / `models get`

| Use case | Recommendation |
|------|------|
| User provides a fuzzy keyword and wants candidates | **`search`** ✓ |
| Find the most recently released models (time-sensitive) | **`search`** (results sorted by update_time DESC) |
| Find by modality / context / capability | **`search`** + corresponding flag |
| Fully enumerate all models | `search` without a keyword (all 152 entries) |
| Exact match by name | `list --name foo` (exact) or `search foo` (includes fuzzy matching) |
| Get complete details for one model (pricing, rate limits, detailed capability descriptions) | `get <name>` |

## References

- [arkcli-models](../SKILL.md) — All models commands
- [arkcli-shared](../../arkcli-shared/SKILL.md) — Authentication and global parameters
