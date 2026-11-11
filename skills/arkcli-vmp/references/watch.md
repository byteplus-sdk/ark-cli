# Multi-endpoint rollout observation

Resolve endpoints, duration and user thresholds. If thresholds are absent, report changes without inventing success criteria. Collect a baseline and record its traffic, coverage and statistical lookback.

```bash
arkcli doctor monitor snapshot --endpoint ep-a,ep-b --window 10m --range 1m --format json
arkcli doctor monitor watch --endpoint ep-a,ep-b --window 5m --range 1m --step 15s --interval 30s --duration 30m --summary-only --format jsonl
```

Save observation.jsonl. Watch collects immediately, then waits interval after each completed collection. Duration bounds the session; window stays independent of interval. Each vmp.watch.v1 line has type=sample with sequence/status/collection, or type=end with completed/cancelled/failed. Completed means observation time elapsed, not healthy. Cancellation preserves earlier lines and emits end with a cancellation error exit. An unfinished current collection is not a full sample. Output failure may prevent the end record; treat missing end as truncated evidence.

Check user criteria per endpoint. Record anomaly time, threshold, metric, coverage and query, then use arkcli-doctor's supported diagnosis. Low coverage or zero traffic cannot pass rollout validation. Overlapping statistical windows mean adjacent observations are not independent events.

Report baseline, observed interval, termination status, anomalous samples, diagnostic evidence, unknowns and next steps. Do not claim persistent monitoring or active alerts.
