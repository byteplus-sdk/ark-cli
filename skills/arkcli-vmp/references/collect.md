# Collection and release comparison

Replace example IDs with resolved resources. Collection returns vmp.collection.v1.

```bash
arkcli doctor monitor collect --endpoint ep-a,ep-b --window 30m --step 1m --range 1m --format json
arkcli doctor monitor collect --endpoint ep-a --start 2026-10-01T00:00:00Z --end 2026-10-15T00:00:00Z --step 1m --range 1m --summary-only --format json
arkcli doctor monitor collect --endpoint ep-a --metric request.rpm.by_api --metric request.duration.quantile --window 1h --range 5m --format json
arkcli doctor monitor collect --endpoint ep-a --metric request.rpm.by_api,request.tpm.input --window 1h --range 5m --format json
```

Supported IDs: request.rpm.by_api, request.error.rate, request.duration.quantile, request.tpm.input, request.tpm.output, quota.purchased.tpm. Quota requires explicit selection. Snapshot summarizes the default first five metrics.

Limits: 1-50 IDs, span up to 744h, step 1s-1h, range 1m-1h, whole seconds, at most two million expected samples. Fixed end is aligned down to start+n*step; inspect actual window. Queries shard at 360 steps and deduplicate overlapping boundaries without widening statistical lookbacks. Endpoint grouping and histogram le are preserved. TPM is converted to tokens/min; RPM is normalized for range.

Save output in new task files without overwriting earlier evidence. Fields:

- workspace_id/endpoints/window/range identify the actual scope.
- results contain endpoint, metric, unit, status, coverage, observed_points, expected_points, peak{ts,value}, latest, sample_mean, and optional points.
- queries contain the actual PromQL, shard interval, status and error. All-query failure prints evidence and exits nonzero; partial collections retain available evidence.
- Missing data has no peak/latest/sample_mean. Summary-only omits points but preserves peak times and provenance.

For comparison, choose equal, nonoverlapping intervals with identical endpoints, metrics, step and range. Start the after interval at least one range after release so the lookback does not cross the change. Compare coverage and traffic magnitude before interpreting per-endpoint peaks, errors and latency. Do not average endpoint ratios into a project error rate. Missing dimensions are inconclusive.

Report the goal, redacted profile scope, inventory source/time, intervals, step/range, units, coverage, peak times, observed changes, evidence paths and limitations.
