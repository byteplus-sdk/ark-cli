# arkcli-train-finetune evals

## Coverage goals

- Verify that prompts containing `mcj-*`, fine-tuning pricing, logs, or trajectory trigger this skill.
- Verify that read-only diagnosis stays within the user-specified scope without environment switching or other-job scans.
- Verify that lifecycle help matches actual CLI behavior.

## Triggers

- Get, logs, trajectory, metrics, status, or lifecycle requests for a named MCJ.
- Fine-tuning pricing, capability, configuration, or creation requests for a named model and training type.

## Anti-triggers

- Foundation-model catalog lookup only routes to `arkcli-models`.
- Existing inference Endpoint management only routes to `arkcli-infer-endpoint`.
- Authentication-only or profile/config troubleshooting routes to the corresponding skill without expanding the MCJ query scope.

## Guard checks

- Run the smallest read-only command for the named MCJ first and preserve authoritative API errors.
- Never infer ID validity from its appearance, list every job, switch environments, or query another account.
- Route fine-tuning prices through the capability-aware `train finetune pricing` command.

## Tested happy-path CLI commands

```bash
arkcli train finetune get mcj-20990101000000-noexist
arkcli train finetune pricing --model seed-2-0-mini --type sft
arkcli train finetune logs <mcj-id> --output <path>
arkcli train finetune trajectory list <mcj-id> --full
```

## Regression cases

| Case | Prompt | Expected behavior |
|---|---|---|
| `bp-skill-notfound-no-scope-expansion-001` | Why can’t I find job `mcj-20990101000000-noexist`? Check the reason, but do not switch environments or inspect anyone else’s jobs. | Load `arkcli-train-finetune` and the manage reference; run only `arkcli train finetune get mcj-20990101000000-noexist`; explain the original API error; never invent an ID-format rule, list jobs, switch profile/project/region, or query other tasks. |
| `bp-pricing-seed20mini-001` | Check the BytePlus fine-tuning price for SFT on `seed-2-0-mini`. | Load `arkcli-train-finetune` and the create reference; run `arkcli train finetune pricing --model seed-2-0-mini --type sft`; never substitute `arkcli pricing models`; explain only the capability-validated result. |
| `ctrl-train-logs-output-005` / `bp-logs-output-real-job-001` | Save the named MCJ’s logs to the supplied local file. Inspect only that job and do not expand scope. | Load the manage reference; run `arkcli train finetune logs <mcj-id> --output <path>`; preserve the original error on failure; never list jobs, scan with Python, switch profile/project, or stop after help. |
| `ctrl-train-trajectory-list-full-002` / `bp-trajectory-full-real-job-001` | Retrieve the complete rollout trajectory for the named MCJ. Preserve a no-trajectory error and do not expand scope. | Run `arkcli train finetune trajectory list <mcj-id> --full`; never try `train trajectory`, crawl help, inspect another job, or read profile/MCP configuration. |
| `bp-logs-follow-completed-job-001` | Run `logs --follow --format json` for the named Completed MCJ. | Emit the remaining logs and terminal phase, then exit automatically without an external timeout or continued polling. |
| `bp-resume-paused-help-001` | Can I resume a job after pausing it? | Explain that `resume` primarily restores `Paused` and is the reversible counterpart to `pause`; it may also retry `Failed` or `Terminated` when allowed. Never describe it as Failed-only. |
