# Failure modes and fallbacks

Route on the structured `error.type`, and on `next_action` for successful calls.
**Never match on error text**, which changes with the interface language.

## Zero hits and search-pipeline failures

| Symptom | Structured signal | Handling |
|---|---|---|
| Search returns an empty `results` | `next_action` states the catalog fallback | Run `arkcli docs list --query "<original query>"`. If that is empty, drop `--query` and inspect the full catalog. Only then may you say the public documentation does not cover it |
| Search action not registered / not routable | `error.type=api`, `error.hint` carries the catalog fallback | Report the request ID and that the search backend is not ready, **and** give the working `docs list --query` + `docs get` path |
| Search backend 5xx | `error.type=service_unavailable`, same `hint` | Bounded retry (at most 2), then handle as the row above |
| Throttled | `error.type=rate_limit` | Back off and retry within bounds; on persistent failure report and fall back to browsing |
| Missing or expired credentials | `error.type=auth`, or the shared gate's `not configured` / `not logged in` | Stop business retries and route to `arkcli-auth`. With existing login authorization, use its two-phase `--no-browser` flow in the same environment, wait for a user code when needed, then resume the original request. Do not change configuration or request retrieval-backend keys |

The core judgement: **an unavailable search is not unreadable documentation.**
`get` and `list` read published artifacts from the product CDN over a separate
pipeline. Never tell the user "no relevant documentation exists" purely because
search failed.

The `--query` fallback above is public-only. On MCP, try plain `docs list`
then `docs get`; if both fail, report MCP unavailability. To continue a filtered
public catalog, keep the original query, snapshot and next_offset together.

A continuation `network` error, including HTTP 404, does not establish a CDN
or offset fault. Compare the actual `--snapshot` with the first successful
response byte-for-byte and correct differences using the saved JSON. Do not
drop the snapshot while retaining the offset. If the exact original value
still fails, report that read failure without claiming a known CDN incident.

## Read errors

| `error.type` | Meaning | Handling |
|---|---|---|
| `not_found` | The document is absent from the active catalog (withdrawn, re-routed, or the search index lags) | Find the current route with `docs list`, or try another result. **Never use `snippet` as the full text.** If no body is retrievable, say so plainly |
| `snapshot_expired` | Republished, a listed document was withdrawn, or **a search snapshot expired or stopped matching the query** (search snapshots live about five minutes) | Discard partial results and restart: for a read drop `--snapshot` and reset `--chunk-start` to 0; for a search drop `--snapshot` and `--offset` and go back to the first page. Do not guess an offset and do not mix revisions or batches |
| `section_not_found` | The anchor ID or title does not exist. **The usual cause is reusing a search result's anchor**: on a page-level hit it is a slug of the document H1, and the published outline excludes the H1 | Run `--outline` for the real ids and pick one, or read the whole page. Also check whether a whole URL was used as the selector or the anchor was decoded manually. **Never invent an anchor, never retry, and there is no silent full-page fallback** |
| `section_ambiguous` | Duplicate heading titles | Use the unique `#anchor-id` instead |
| `section_unavailable` | Published heading hierarchy cannot be mapped onto the Markdown body (common on DSL-dense pages) | Explain that per-section reading is unsupported on this page and read the full page |
| `validation` | Out-of-range argument, conflicting selectors, MCP capability difference, or no configured backend | Follow the capability table in commands.md. Public continuation keeps its snapshot; MCP starts with chunk parameters. Never mix partial content from different backends or revisions |
| `invalid_response` | Invalid search/catalog target, revision or shape, body checksum/encoding, or heading metadata | Report it as a backend anomaly; do not loop retries or read unverified content |

The public backend automatically evicts an invalid cache entry and fetches it
once. Invalid downloads are never cached. If `invalid_response` persists, no
manual cache cleanup is needed; rerun normally after the upstream is fixed.

## Degradation caused by custom DSL

Published Markdown keeps the documentation site's custom tags (`<Card>`,
`<Columns>`, `<ColumnsItem>`, `<Tabs>`, `<Tab>`, `<Note>`, `<Tip>`, `<Warning>`,
`<Danger>`, `<APILink>`, `<Attachment>`, `<RenderMd>`, `<span id>`, and inline
links carrying a trailing attribute block such as `{target="_self"}`). The CLI
does not downgrade them.

Consequences and handling:

- **`--section` can fail**: headings inside custom tags never reach the Markdown
  AST, which surfaces as `section_unavailable`. Read the full page instead.
- **Chunk boundaries can cut a DSL block**: a single oversized block is split.
  Concatenate consecutive `content` values before interpreting, and never judge
  completeness from half a tag.
- **Tags are not content**: quote only the text and fenced code blocks inside
  them. Your answer should never contain `<Card>` or `<Tab>` markup.

## Bounded retry rules

- Network-class errors (`network`, `service_unavailable`, `rate_limit`): at most 2 backoff
  retries.
- `auth`, `not_found`, `validation`, `invalid_response`, `section_*`: **never retry**; they will not
  change on a second attempt.
- Never loop `get` on the same missing route, and never guess a hash or CDN URL
  to retrieve withdrawn content.
