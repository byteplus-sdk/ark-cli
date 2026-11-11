# Two-week peaks and capacity evidence

Fix a fourteen-day interval and use one-minute step/range. The CLI shards and deduplicates queries without replacing historical minute peaks with coarse averages.

```bash
arkcli doctor monitor capacity --endpoint ep-a,ep-b --start 2026-10-01T00:00:00Z --end 2026-10-15T00:00:00Z --step 1m --range 1m --format json
```

vmp.capacity.v1 includes RPM, input TPM, output TPM and purchased completion TPM collection evidence per endpoint, peak timestamps and coverage. Missing data does not become zero. Units are req/min or tokens/min.

Status is inconclusive because purchased completion quota and total output traffic have unverified billing scope. You can report observed peaks, times and coverage; this evidence alone does not establish a safe quota reduction. Do not sum independent endpoint peaks as a simultaneous project peak or divide input TPM by completion quota.

For a quota-change request, prepare recommendations and missing facts using the current product's supported capabilities. Do not invent a BytePlus quota rebalance command. Capacity decisions also require forecast traffic, headroom policy, throttling evidence and a comparable quota scope.

Report per-endpoint peaks/times, coverage, failed or empty shards, quota evidence, limitations and facts needed to decide. E2E latency, throughput and quota are different metrics; TTFT remains unavailable.
