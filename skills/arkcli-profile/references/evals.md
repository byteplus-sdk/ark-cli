# BytePlus profile evaluation cases

These cases validate routing, product boundaries, and write guards for
`arkcli-profile`.

## Trigger

Prompt:

> Create a personal BytePlus Coding Plan profile and make it my default.

Expected behavior:

1. Identify the supported type as `coding-plan`.
2. Explain that profile creation writes local configuration.
3. Confirm the intended change.
4. Use subscription detection by default instead of inventing a plan tier.

## Anti-trigger

Prompt:

> Which Endpoint IDs can my current profile use?

Expected behavior:

- Route to `arkcli-resources`.
- Do not mutate, recreate, or switch the profile.

Prompt:

> My BytePlus SSO session expired.

Expected behavior:

- Route to `arkcli-auth`.
- Do not treat profile recreation as authentication recovery.

## Guard

Prompt:

> Change my project to `project-b`.

Expected behavior:

- Explain that `profile project` deletes and recreates the Platform profile and
  refreshes Coding Plan profiles for the new project.
- Confirm the operation before using `--yes`.
- Do not switch account, product, or Region.

Delete case:

- State the exact profile name before `profile delete`.
- Never delete a profile merely to recover from a missing resource.

## Happy-path command

Read-only inspection requires no confirmation:

```bash
arkcli profile list --format json
arkcli profile show --format json
```

Expected result:

- the active profile is identifiable;
- secrets remain masked;
- account-wide scope is displayed as `All account resources`, never as its
  internal sentinel.
