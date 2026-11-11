# Endpoint snapshot

```bash
arkcli doctor monitor snapshot --endpoint ep-a,ep-b --window 5m --range 1m --format json
```

Snapshot is a summary-only collection of five default metrics with fixed shared query bounds, endpoint dimensions, peak times, coverage and queries. Read the [collection contract](collect.md). Available is a data status; no traffic cannot verify latency or error rate.
