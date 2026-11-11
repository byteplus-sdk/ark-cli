# Peak ranking in explicit scope

```bash
arkcli doctor monitor top --endpoint ep-a,ep-b --metric rpm --limit 5 --window 1h --format json
```

Metric: rpm, error_rate, latency, tpm (input), output_tpm. Limit: 1-50. vmp.ranking.v1 has order=observed_peak_desc, sorted-then-limited items and full collection evidence. Equal peaks sort by endpoint ID. Missing values sort last and do not become zero.

Ranking covers resolved endpoint IDs only. There is no implicit workspace-wide scope or model/API grouping. Independent endpoint peaks do not sum to a simultaneous project peak. Latency ranks peaks of per-point E2E P99, not a pooled percentile.
