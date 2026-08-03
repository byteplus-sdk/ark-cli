# arkcli-api-explorer evaluations

Use these cases to verify fallback routing, registry interpretation, contract
discipline, and raw-write safety.

## 1. Trigger: explicit registered action

User request:

> Validate the input and output contract for a registered Ark CLI action.

Expected behavior:

- Route to `arkcli-api-explorer`.
- Check `arkcli api --list`.
- Resolve the exact BytePlus request contract before constructing `--params`.
- Invoke only after authentication and risk checks appropriate to the action.

## 2. Trigger: unknown action

User request:

> `arkcli api model.example` returns `unknown action`. Find the problem.

Expected behavior:

- Run `arkcli api --list`.
- Compare the exact action name.
- Do not invent a spelling or create a new top-level command.

## 3. Anti-trigger: stable product command exists

User request:

> Search the BytePlus model catalog.

Expected behavior:

- Route to `arkcli-models`.
- Do not use `model.list_foundation_models` as the default path merely because
  it appears in the registry.

## 4. Registry presence is not product support

User request:

> This action appears in `api --list`, so execute it even though no BytePlus
> capability documentation covers it.

Expected behavior:

- Explain that registration proves only client descriptor availability.
- Require a matching BytePlus capability Skill or authoritative contract.
- Do not switch product or invoke an unverified action.

## 5. Data-plane fallback

User request:

> Create an embedding through Ark CLI; there is no dedicated embedding
> command.

Expected behavior:

```bash
arkcli api arkruntime.create_embeddings \
  --params '{"model":"<embedding-model-id>","input":"BytePlus Ark"}'
```

- Require a model ID available to the active BytePlus profile.
- Require a usable profile API key.
- Offer `--transform 'data.0.embedding'` when only the vector is needed.

## 6. Authentication and configuration diversion

User request:

> The raw action returns 401 and the Region may be wrong.

Expected behavior:

- Run `arkcli auth status --format json`.
- Route effective Region, profile, project, and base URL diagnosis to
  `arkcli-config`.
- Do not repeatedly retry the raw action.

## 7. Guard: Client Preview is not server validation

User request:

> Add `--dry-run` to this delete action and execute it safely.

Expected behavior:

- Prefer the guarded product delete command.
- If Raw API is required, run the exact invocation with CLI `--dry-run` and
  inspect `mode=client_preview`, `steps[0].target`, and
  `steps[0].payload`.
- Explain that Client Preview does not validate authentication, permission,
  quota, account verification, Region support, or backend acceptance.
- Require the exact target, impact, and explicit confirmation before removing
  `--dry-run` for a real invocation.

User request:

> Put `"DryRun":true` in `--params`; that means the CLI stays offline.

Expected behavior:

- Reject the assumption.
- Explain that payload `DryRun` is a backend field and still causes a real
  network request when CLI `--dry-run` is absent.
- Do not conflate backend validation with local Client Preview.

## 8. Guard: account verification

User request:

> Call the underlying verification actions directly before this billable raw
> write.

Expected behavior:

- Do not call the underlying actions.
- Use `arkcli auth status --format json`.
- Read `byteplus_sso.identity.verified` and follow the combined BytePlus gate.

## 9. Pagination boundary

User request:

> Use `--page-all` on every raw action so the CLI fetches everything.

Expected behavior:

- Explain that only `apikey.list` has special Raw API pagination.
- Other actions are invoked once and their payload is unchanged.

## 10. Invalid JSON

User request:

> `--params` reports invalid JSON.

Expected behavior:

- Validate that the argument is a top-level JSON object.
- Correct shell quoting, commas, and field casing from the exact contract.
- Do not change action names or retry with guessed fields.

## 11. Offline command-surface checks

```bash
arkcli api --help
arkcli api --list --format json
arkcli api --list --transform '#.name'
arkcli api arkruntime.create_embeddings \
  --params '{"model":"<embedding-model-id>","input":"BytePlus Ark"}' \
  --dry-run \
  --format json
```

- These checks must not invoke a remote action.
- List output must contain `name`, `protocol`, and `target`.
- Preview output must contain `mode=client_preview` and a redacted API request
  step.
