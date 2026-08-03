# `plans team rotate-apikey`

Read [`../SKILL.md`](../SKILL.md) and
[`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first.

This is a destructive credential write. Every successfully rotated key becomes
invalid immediately, so confirm that all affected applications are ready to
receive their replacement keys before execution.

## Syntax

```bash
arkcli plans team rotate-apikey \
  --seat-ids seat-001,seat-002 \
  [--yes]
```

BytePlus supports only the explicit administrator path:

- `--seat-ids` is required and accepts a comma-separated list.
- Omitting `--seat-ids` is a validation error; BytePlus does not perform an
  implicit self-seat lookup.
- The backend resolves the Coding Plan Team subscription from each SeatID.

## Safe workflow

1. Identify every application using the affected keys.
2. Prepare the approved secret destination and replacement rollout.
3. Verify the exact scope with the read-only seat list:

   ```bash
   arkcli plans team seat-list --plan coding-plan-team
   ```

   `rotate-apikey` does not register `--dry-run` because the final workflow
   depends on live subscription and credential state.
4. Warn the user that old keys become invalid immediately and obtain explicit
   confirmation.
5. Execute the real rotation:

   ```bash
   arkcli plans team rotate-apikey \
     --seat-ids seat-001,seat-002 \
     --yes
   ```

## Output and partial failure

The structured result contains `success_count`, `failed_count`, `success`, and
`failed`. A batch may partially succeed. Inspect both arrays even when the
command exits nonzero.

`success[].api_key` is the new plaintext key. Never paste, log, or persist it
in chat. Move it directly to the approved secret destination, then refresh the
affected local profile if it still references the invalidated key:

```bash
arkcli profile keys refresh --profile <profile-name>
```

Report only masked key suffixes, SeatIDs, counts, and failure reasons.
