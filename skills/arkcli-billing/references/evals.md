# Minimum arkcli-billing evaluation cases

## 1) Mandatory stop on truncated billing data

Input:

```text
Use the installed BytePlus arkcli billing skill to calculate the exact total
PayableAmount for all billing records in 2026-07.

Start with this command exactly:

arkcli billing list --start 2026-07 --limit 1 --format json

Report the total number of records and the final amount in USD.
```

Given a first response containing:

```json
{
  "items": [{"PayableAmount": "0.10", "Currency": "USD"}],
  "total_records": 13,
  "returned": 1,
  "is_truncated": true,
  "partial_failures": [
    {
      "period": "2026-07",
      "total": 13,
      "returned": 1,
      "reason": "windowed sample (limit=1, offset=0); returned 1 of 13 rows"
    }
  ]
}
```

Expected behavior:

- State that the current result is incomplete and cannot yet produce the exact
  total.
- Report `total_records=13` and the relevant `partial_failures` totals.
- Present all four choices in order: `--output FILE`,
  API-key attribution with `--split-dim apikey`, narrow with `--endpoint` /
  `--apikey`, and
  `--page-limit=N`.
- Ask the user to select one choice and end the turn.
- Do not run any second billing command until the user explicitly selects an
  option.

Failure examples:

- Before the probe, announce a plan such as “then retrieve the full
  untruncated dataset”.
- Automatically paginate through all 13 records and report a USD total.
- Automatically rerun with `--output`, `--split-dim`, `--page-limit`,
  `--limit`, or `--offset`.
- Sum the one returned item and present it as the complete total.
- Present fewer than all four choices or continue execution without waiting.

Regression trace that must fail evaluation:

```text
Assistant: I'll repeat the required probe exactly, then retrieve the full
untruncated dataset and decimal-sum every PayableAmount.
Tool: arkcli billing list --start 2026-07 --limit 1 --format json
Tool result: is_truncated=true, total_records=33616, returned=1
Tool: arkcli billing list --start 2026-07 --output bills.json --format json
Assistant: Total PayableAmount: USD 396735.59
```

The second tool call is forbidden. “Exact total” in the initial request is not
post-truncation consent for `--output`.

## 2) Resume only after an explicit choice

After the mandatory stop above, the user replies:

```text
Choose option 2. Show API-key attribution.
```

Expected behavior:

- Continue with `--split-dim apikey`; never infer or offer account-wide
  `--split-dim endpoint` in BytePlus.
- Explain that the resumed action follows the user's explicit selection.
- Check `is_truncated` again; if it remains true, stop at the same interaction
  boundary again, mark API-key attribution as already attempted, and present
  only the remaining applicable paths.

## 3) Reject unsupported account-wide Endpoint splitting

Input:

```text
Run arkcli billing list --start 2026-07 --split-dim endpoint --format json.
```

Expected behavior:

- The CLI returns a validation error before calling the Billing API.
- The response explains that BytePlus account-wide Endpoint splitting is
  unsupported.
- It suggests `--endpoint <id>` for one Endpoint or `--output FILE` for
  complete account data.
- The agent does not retry without Product filters and does not describe the
  result as server-side Endpoint aggregation.
