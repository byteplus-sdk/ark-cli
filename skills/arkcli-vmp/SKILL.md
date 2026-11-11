---
name: arkcli-vmp
description: Orchestrate ARK observability tasks including multi-endpoint rollout observation, before/after release comparison, two-week traffic peaks and capacity evidence. Use when asked to watch a rollout, compare a release, find busy endpoints or review historical capacity. Combine resource inventory, VMP collection and diagnosis into an evidence-backed report. Alert writes and scheduling are unavailable.
---

# ARK observability tasks

Read and follow `../arkcli-shared/SKILL.md` first.

Complete the user's investigation or observation. The CLI provides reliable queries, aligned time ranges, sharding, statistics and structured output. This Skill chooses scope, sequence, branches, evidence artifacts and completion criteria. All collection commands live under `arkcli doctor monitor`; single-endpoint diagnosis remains `arkcli doctor infer-endpoint`.

## When To Trigger and References

| Goal | Workflow | Reference |
|---|---|---|
| Observe a rollout | Resolve endpoints and criteria, collect baseline, observe for a bounded duration, diagnose anomalies, report | [Observation](references/watch.md) |
| Compare a release | Fix comparable before/after intervals, collect identical metrics, check coverage, compare each endpoint | [Collection](references/collect.md) |
| Review two-week capacity | Collect minute-resolution history, locate peaks and gaps, verify quota semantics, report limitations | [Capacity](references/capacity.md) |
| Find current anomalies or busiest endpoints | Resolve inventory, collect summary or ranking, drill into evidence | [Snapshot](references/snapshot.md), [Ranking](references/top.md) |
| Configure alerts or daily checks | Explain the backend boundary and prepare an implementation proposal | [Alerts](references/alert.md) |

## When NOT To Trigger

Route a simple token usage or invoice question to arkcli-usage or arkcli-billing. Use arkcli-doctor for a known failure needing diagnosis, and deployment/endpoint Skills for resource creation or mutation. Do not expand those tasks into monitoring just to use this Skill.

## Execution protocol and Guard Checklist

1. Establish profile, explicit endpoint IDs, time range and user criteria. Use supplied IDs directly. For a whole project, read `../arkcli-infer-endpoint/SKILL.md`, resolve every control-plane inventory page for the selected profile, then pass explicit IDs. A shared workspace is not project scope. Save inventory and resolution time. Each batch supports up to 50 IDs; multiple batches must share fixed start/end.
2. When scope needs verification, follow shared and run `arkcli auth status` / `arkcli auth whoami` separately. Inspect their original results and save only redacted product, region and project information; mark missing scope fields as unverified. Do not use `profile show` as a routine read-only identity check because it may sync keys and write configuration. Collection resolves an existing workspace binding without creating one. For absent or expired credentials, follow `../arkcli-auth/SKILL.md`. Missing permissions/bindings are incomplete prerequisites; report the actual error and next step.
3. Use fixed RFC3339 whole-second start/end for history and comparisons. Window is query span, step is sample spacing, range is statistical lookback per point. They are independent. Fourteen days is `336h`, not `14d`. See the collection reference.
4. Save JSON/JSONL in a new task directory alongside the user goal, inventory, commands and report. The CLI owns authentication, queries, sharding, deduplication and unit conversion. Do not replace these with temporary scripts that change peak semantics.
5. Inspect collection.status, result status/coverage and queries. Available means finite samples cover the requested grid; it is neither proof of complete source ingestion nor a health verdict. Partial/no_data/failed must not become zero. Observed zero RPM alone cannot establish latency or error-rate success.
6. For anomalies, read `../arkcli-doctor/SKILL.md` and diagnose the affected endpoint through its supported entry points. Separate observations, hypotheses, diagnostic evidence and recommendations. Resource or alert mutations require the user's authorization.
7. Deliver a verdict (criteria met, criteria violated, or inconclusive), per-endpoint evidence, units, intervals, coverage, peak times, gaps and next steps. Without user thresholds, report descriptive findings instead of inventing a rollout pass.

## Boundaries

- Default metrics: RPM, error rate, request E2E P99, input/output TPM. E2E P99 is not TTFT; no verified TTFT collection contract exists.
- sample_mean averages observed points. It is not a request-weighted error rate. A peak of per-point P99 values is not the P99 of all requests over two weeks.
- Capacity returns evidence and inconclusive: purchased completion TPM and total output traffic have unverified billing scope. Do not divide them to recommend a utilization or quota change.
- Watch is bounded foreground observation with JSONL samples and an end record. It stops when interrupted; persistent alerts, schedules and resume are not implemented.
- No `doctor monitor workspace list`, `--since`, `--output`, `--once`, or `--threshold`. Use `--workspace-id`, `--window` or `--start --end`, and global `--format`. Read-only collection has no `--dry-run`.
