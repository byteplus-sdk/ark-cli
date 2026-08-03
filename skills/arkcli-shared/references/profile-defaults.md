# Profile resource defaults

Read this reference only when chat, generation, or deployment needs a default
model or Endpoint. Ordinary read-only resource queries do not need it.

Profiles may store a default resource per modality:

- `text`
- `image`
- `video`

The default may be a model ID or an Endpoint ID depending on the active
profile and the capability contract. Never infer one form from the other.

## Selection rules

1. If the user passes `--model` or `--endpoint`, use the explicit value after
   validating that the active BytePlus capability supports it.
2. If the user omits the value, let Ark CLI resolve the profile default.
3. If Ark CLI reports that no default exists, list compatible resources first,
   then ask the user which one should become the default.
4. Do not silently select a resource from another Region, project, account, or
   product.

## Drift prompt

After a successful command that used an explicit resource different from the
current default, offer to promote it:

> This command used `<new>`, while the current `<modality>` default is `<old>`.
> Set `<new>` as the new default?

Only update the profile after the user confirms. Do not prompt after a failed
command, when the command already used the default, or when the user has
declined the same suggestion in the current session.
