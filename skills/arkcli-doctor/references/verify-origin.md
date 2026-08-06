# ModelArk Generation-Origin Verification

## Scope

Use:

```bash
arkcli doctor +verify-origin <media-url> [media-url...] --format json
```

The command checks whether 1-20 public image or video URLs contain technical
features associated with the ModelArk generation process.

It does not determine content truthfulness, copyright, ownership, legal
certification, content safety, compliance, media quality, or playability.

## One Confirmation for the Entire Batch

Run the complete batch once without `--yes`:

```bash
arkcli doctor +verify-origin \
  "https://example.com/a.mp4" \
  "https://example.com/b.png" \
  --format json
```

This call does not acquire the origin-verification invoker or call Create/Get.
It returns one disclosure covering the entire batch:

- 20 free calls per account, but remaining quota is unknown;
- USD 0.015 per successful call after the free quota;
- successful `True`, `False`, and `Null` results are billable;
- authentication and rate-limit failures are not billable;
- daily limit 1000;
- Create and Get target 20 QPS.

Show the disclosure and wait for explicit user confirmation. Then rerun the same
batch with one `--yes`:

```bash
arkcli doctor +verify-origin \
  "https://example.com/a.mp4" \
  "https://example.com/b.png" \
  --yes \
  --format json
```

That single `--yes` authorizes every URL in the current batch. Never ask once
per URL.

## Execution and Recovery

The CLI owns:

```text
CreateArkOfficialResultQuery
  -> Result.QueryID
  -> GetArkOfficialResult every 5 seconds
  -> running continues
  -> succeeded or failed terminates
```

Do not launch separate CLI processes, write a shell loop, call raw Actions, or
poll in the Agent.

Resume existing tasks without creating or reconfirming them:

```bash
arkcli doctor +verify-origin \
  --query-id "<query-id-1>" \
  --query-id "<query-id-2>" \
  --format json
```

Each URL may be created at most once. A Get retry must preserve its QueryID and
must never return to Create.

## Authentication

This is a control-plane OpenTOP workflow:

- use BytePlus login-derived STS or configured AK/SK;
- do not use an Ark data-plane API key;
- do not pass global `--api-key` or `--base-url`;
- do not retry through another product.

## Exact Output Handoff

The final response must contain only the complete CLI stdout JSON:

- no summary, interpretation, or translation;
- no extraction of only `Result` or `Message`;
- no Markdown code fence, prefix, or suffix;
- no `--transform`;
- no YAML, table, CSV, or JSONL.

Never convert:

- `True` into legal or official certification;
- `False` into certainty that ModelArk was not involved;
- `Null` into `False`.
