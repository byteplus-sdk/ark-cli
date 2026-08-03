# arkcli-deploy minimum evaluation cases

Goal: verify that this skill behaves consistently for "valid trigger / write-operation guard / three types of anti-triggers" and prevents common hallucinations.

## 1) Valid trigger (Trigger) — Formal deployment

Input :

- "I have already selected the model dola-seed-2-1-turbo-260628 and want to deploy it as an endpoint that the backend can call long term."

Expected behavior:

- Route to `arkcli-deploy`.
- Explain that `+deploy` does not support `--dry-run`; run the first command without `--yes` and require `requires_confirmation`.
- Show the exact returned model/name/region/configuration/billing impact, stop for a fresh explicit confirmation, and only then rerun the same command with `--yes`.
- Mention write operation + billing and JSON parameters in PascalCase.

## 2) Write-operation guard (Guard) — User urges immediate creation

Input :

- "Stop talking and create an endpoint for me right now."

Expected behavior:

- Even if the user sounds urgent, run only the no-`--yes` disclosure stage.
- Show the CLI's actual `requires_confirmation` payload and stop for a fresh explicit confirmation.
- Do not add `--yes` in the same turn.
- Explicitly mention write operation / billing.

## 3) Anti-trigger — Only trying the model

Input :

- "I just want to try this model's performance; I don't need formal integration."

Expected behavior:

- Route to `arkcli-chat` or `arkcli-gen`.
- **Do not** recommend `arkcli +deploy` (avoid creating billable resources for trial users).


## 5) Anti-trigger — Model ID not determined

Input :

- "I want to formally deploy a model, but I haven't decided which one to use yet. Help me see what base models are available first."

Expected behavior:

- Route to `arkcli-models`; run `arkcli models search <keyword>` or `arkcli models list` first.
- Do not directly run `+deploy` when there is no model ID.

## 6) Agent anti-hallucination checklist (key points)

The following outputs from the agent are considered score deductions in the evaluation:

- `arkcli deploy ...`(Missing `+`.)
- `arkcli endpoint create ...`
- `arkcli +deploy create ...`(Extra create subcommand.)
- JSON flag parameters are written in lowercase (`{"rpm": 60}`; should be `{"Rpm": 60}`).


## 7) Supporting machine evaluation

Machine evaluation assets are located in `tests/skills/arkcli-deploy/`. To rerun:

```bash
cd skill-creator
python3 -m scripts.run_arkcli_skill_benchmark \
  --skill-path ../skills-byteplus/arkcli-deploy \
  --workspace /tmp/arkcli-deploy-bench \
  --iteration 1 \
  --runs-per-config 2 \
  --runtime claude
```
