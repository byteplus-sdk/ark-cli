# Troubleshooting and capability boundaries

Read this reference when a command fails and the failure has not yet been
classified.

## Failure routing

1. Classify the failure before retrying:
   - authentication or expired credentials;
   - runtime override or profile configuration;
   - invalid arguments;
   - missing model, Endpoint, or other resource;
   - unsupported BytePlus capability;
   - missing external dependency or permission.
2. For authentication failures, run `arkcli auth status --format json` and use
   the `arkcli-auth` Skill.
3. For profile, Region, project, API key, or base URL confusion, compare
   effective runtime status with `arkcli profile show --format json`.
4. For an unknown resource identifier, use the corresponding list or search
   command instead of guessing.
5. If the product command does not cover the task, inspect the installed
   BytePlus command help and the matching BytePlus capability Skill before
   considering the raw API explorer.

## Error meaning

Keep these outcomes distinct:

- the product does not support the capability;
- the current account lacks permission;
- the resource does not exist or is outside the active scope;
- the Region is unsupported;
- an external service, SDK, or entitlement is not enabled.

Do not reinterpret an authorization error as proof that the product lacks the
capability. Do not reinterpret a missing BytePlus command as a permission
issue.

## Retry rules

- Retry only operations documented as safe and idempotent.
- Refresh authentication once when the status explicitly reports expiry.
- Do not loop over business commands after repeated 401, 403, or scope errors.
- For writes, preserve the original request and ask for confirmation before a
  retry that could create duplicate resources or additional cost.
