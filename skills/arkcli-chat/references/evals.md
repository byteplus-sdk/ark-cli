# Chat regression scenarios

This is the readable acceptance contract. Executable cases and critical gates
belong to arkcli-eval-kit. Record the CLI ref, installed Skill hashes, Agent and
model, selected profile, actual tool calls and delivered files. Listing a case
here does not claim online coverage; missing credentials, skipped or simulated
cases do not count as passes. Do not reuse another product's credentials or
Regions for these scenarios.

| Scenario | User request or fixture | Required evidence |
| --- | --- | --- |
| Trigger and defaults | Use the current default without changing configuration. | Load Chat, use the authorized callable ID, do not edit the default or substitute a host-generated answer. |
| Anti-trigger | Generate a picture, or transcribe audio to timestamped subtitles. | Route to Gen or Understand respectively. |
| Admission | Missing/revoked Key, 403, or quota/rate-limit fixture. | Distinguish failures; no automatic rotation, account/profile switch or billing-lane fallback. |
| Preview options | Preview temperature zero, at most 64 tokens, no storage, prefix cache disabled. | Real local dry-run; temperature=0, max_output_tokens=64, store=false, caching.prefix=false; no business request. |
| Optional combinations | Omission; prefix cache plus token cap; function tools plus max-tool-calls. | Preserve omission versus explicit zero/false; reject at existing guards instead of silently removing user constraints. |
| Strict JSON | Two-day itinerary with only city, days and exactly two items, using a supplied schema. | One completed real response, direct JSON in content, schema validation, saved file exactly matches that response; no extra successful request just to save or extract fields. |
| Stored continuation | Remember a random code, then ask for it without repeating it. | Stored first response and its real ID in the second call; no hidden code in the second prompt, no fresh conversation masquerading as continuation. |
| Streaming | Stream the reply and save only the final answer. | Handle actual event names and terminal state; saved answer matches response text, not reasoning or event JSON. |
| Multimodal | Compare two pictures. | Both inputs belong to this call; no unapproved cross-profile visual-model fallback. |
| Numbers and secrets | Schema const=9007199254740993 and a URL with synthetic credentials. | Exact final JSON number; redact nested/raw URL userinfo and query credentials without losing safe fields. |
| Local delivery failure | A successful response exists but extraction or writing fails; an output file already exists. | Reuse captured output and repair only local delivery; no new inference or unapproved overwrite. Distinguish this from requested regeneration or continuation. |

Run preview checks with networking and filesystem writes blocked. Existing
preview acceptance of malformed schema, missing assets and NaN/Inf is unchanged;
do not confuse it with real-request admission. Compare original JSON numbers,
not two float64 values that have already lost precision.

Calibrate the evaluator with a known nonempty success before testing wrong IDs,
missing terminal events, empty answers, malformed JSON, changed files and Agent
success claims without corresponding tool results. Online tests require an
authorized BytePlus account and admitted model; domestic results are not
evidence of BytePlus online coverage.
