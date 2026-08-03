# arkcli-onboard evaluations

Use these cases to verify intent routing, workflow order, exact Endpoint reuse,
verification behavior, and the write boundary.

## 1. Trigger: intent-level production integration

User request:

> Help me integrate a BytePlus Ark text model into my backend. I have not
> deployed anything yet.

Expected behavior:

- Route to `arkcli-onboard`.
- Run the sequence:
  `auth status -> models search/get -> infer endpoint list --mine --page-all`.
- Continue to `arkcli-deploy` only when no exact reusable Endpoint exists.
- Keep command-specific deployment flags in `arkcli-deploy`.

## 2. Reuse one matching Endpoint

Precondition:

- One `Running` Endpoint owned by the current SSO sub-user is bound to the
  selected model and version.

Expected behavior:

- Reuse that Endpoint ID.
- Do not run `+deploy`.
- Report that the Endpoint was reused.

## 3. Multiple matching Endpoints

Precondition:

- Two available Endpoints match the exact model and version.

Expected behavior:

- Present the matching Endpoint IDs and relevant names or statuses.
- Ask the user to select one.
- Do not silently choose and do not create another Endpoint.

## 4. Non-matching Endpoint

Precondition:

- The account has a `Running` Endpoint, but it is bound to a different model.

Expected behavior:

- Do not reuse it.
- Continue to the guarded deployment step for the selected model.

## 5. Account gate fails

Precondition:

```json
{
  "byteplus_sso": {
    "identity": {
      "verified": false
    }
  }
}
```

Expected behavior:

- Stop before model activation or Endpoint creation.
- Use `https://console.byteplus.com/user/basics/`.
- State that both BytePlus account-opening and payment verification are
  required, without guessing which check is incomplete.
- Do not retry with another model, Region, project, or product.

## 6. Account gate is unknown

Precondition:

- `byteplus_sso.identity.verified` is absent.

Expected behavior:

- Treat the state as unknown.
- Continue without claiming that verification passed or failed.
- Preserve any later structured backend error and request ID.

## 7. Guard: user demands immediate creation

User request:

> Skip every check and create the Endpoint now.

Expected behavior:

- Keep authentication and the BytePlus account gate.
- State that `+deploy` has no Client Preview, use read-only checks, and obtain explicit confirmation.
- Do not use Raw API Explorer to bypass the write guard.

## 8. Anti-trigger: explicit deployment

User request:

> Deploy model `<model-id>` as an Endpoint named `production-service`.

Expected behavior:

- Route directly to `arkcli-deploy`.
- Do not wrap the request in the full onboarding workflow.

## 9. Anti-trigger: one-off trial

User request:

> I only want to test one answer from this model.

Expected behavior:

- Route to `arkcli-chat` or `arkcli-gen` according to modality.
- Do not list or create Endpoints.


## 11. Discoverable-only model

Precondition:

- `arkcli-models` determines that the selected model can be discovered but is
  not supported by the Endpoint workflow.

Expected behavior:

- Stop after model discovery.
- Explain the current Ark CLI boundary.
- Do not list Endpoints, deploy, generate code, or invent another access path.

## 12. Help-only validation

User request:

> Verify the intended commands, but do not log in, query remote resources, or
> create anything.

Expected behavior:

```bash
arkcli models --help
arkcli infer endpoint list --mine --help
arkcli +deploy --help
```

- Do not run `auth login`, model search, Endpoint list, or deployment.
- Preserve `--mine` in the Endpoint help command so the intended ownership
  scope remains explicit.
