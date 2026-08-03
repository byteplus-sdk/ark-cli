---
name: arkcli-profile
version: 0.2.0
description: "Manage product-isolated BytePlus profiles: inspect and select profiles, create supported profile types, re-select project scope, synchronize API keys and Coding Plan models, and choose default resources."
metadata:
  requires:
    bins: ["arkcli"]
  cliHelp: "arkcli profile --help"
---

# BytePlus profiles

Read [`../arkcli-shared/SKILL.md`](../arkcli-shared/SKILL.md) before using this
Skill.
The entire profile domain manages local identity state and rejects
`--dry-run`; inspect with `show/list` and obtain explicit confirmation for
mutations.

A profile is a local BytePlus configuration slice that binds a profile type,
the fixed BytePlus Region, project scope, identity, API keys, and default
resources. Selecting a profile does not switch product or tenant.

## Supported profile types

| Type | Purpose |
|---|---|
| `platform` | Pay-as-you-go Platform Endpoint and inference usage. |
| `coding-plan` | Personal Coding Plan usage. |
| `coding-plan-team` | Coding Plan Team usage through an assigned seat. |

Do not suggest another profile type. The BytePlus Region is fixed to
`ap-southeast-1`.

## When To Trigger

- listing profiles or identifying the active profile;
- switching the local default profile;
- creating or deleting a supported BytePlus profile;
- changing the active project scope;
- synchronizing or selecting profile API keys;
- inspecting Coding Plan model defaults;
- selecting a default text, image, or video resource.

## When NOT To Trigger

Use [`../arkcli-auth/SKILL.md`](../arkcli-auth/SKILL.md) for login and credential
health. Use [`../arkcli-resources/SKILL.md`](../arkcli-resources/SKILL.md) to
list resources currently visible to a profile.
Do not use profile mutation as a fallback for a missing resource, permission
error, or unsupported capability.

## Profile resolution

The effective profile is selected in this order:

```text
--profile
  > ARK_PROFILE
  > config default_profile
  > first valid platform profile
  > default sentinel
```

For commands that accept `--profile <name>`, Ark CLI uses that profile's
identity and configuration for the operation. It does not merely change the
name of the output target.

## Project scope

BytePlus can operate in a named project or across the account:

- `All account resources` is the user-facing account-wide option;
- account-wide profile names use the `accountwide` segment, for example
  `platform_ap-southeast-1_accountwide`;
- the account-wide internal sentinel is an implementation detail and must not
  be copied into commands or output explanations.

First login selects account-wide by default. Manual `profile create` keeps its
documented `default` project fallback unless the user selects or supplies
another project.

`arkcli profile project [<project-name>]` changes project scope without
reauthenticating, but it is not a simple field update:

```text
existing platform profile
  |
  +-- delete
  |
  +-- create a fresh platform profile for the selected project
  |
  +-- refresh Coding Plan subscription and team-seat profiles
        |
        +-- update or create profiles available in the new project
        +-- remove profiles not available in the new project
```

The command requires confirmation and does not support `--dry-run`. Without a
positional project name, it loads the selectable BytePlus projects and includes
`All account resources`.

## Command map

| Command | Behavior |
|---|---|
| `arkcli profile list` | List local profiles and identify the default. |
| `arkcli profile show [profile]` | Show one profile with masked secrets and the derived data-plane URL. |
| `arkcli profile use [profile]` | Change the local default profile. |
| `arkcli profile create` | Create `platform`, `coding-plan`, or `coding-plan-team`. |
| `arkcli profile delete <profile>` | Delete a local profile; use `--yes` only after confirmation. |
| `arkcli profile rename <profile> --to <display-name>` | Change only the display label, not the internal profile key. |
| `arkcli profile project [<project-name>]` | Replace the Platform project slice and refresh Coding Plan profiles. |
| `arkcli profile keys list|use|refresh` | Inspect, select, or synchronize profile API keys. |
| `arkcli profile models list|refresh` | Inspect or refresh Coding Plan model defaults. |
| `arkcli profile set-default <id>` | Save a default resource for one modality. |

## Guard Checklist

1. Start uncertain profile work with:

   ```bash
   arkcli profile show --format json
   arkcli profile list --format json
   ```

2. Before `create`, `delete`, or `project`, state the exact local configuration
   impact and obtain confirmation.
3. Before `use`, `rename`, `keys use`, `keys refresh`, `models refresh`, or
   `set-default`, state the target profile.
4. Use `--plan-tier lite|pro` only for a personal `coding-plan` profile when
   the user explicitly needs to override subscription detection.
5. Do not edit `~/.arkcli-bp/` or `config.yaml` directly.

## References

- Create a profile:
  [`references/arkcli-profile-create.md`](references/arkcli-profile-create.md)
- Manage API keys:
  [`references/arkcli-profile-keys.md`](references/arkcli-profile-keys.md)
- Select a default resource:
  [`references/arkcli-profile-set-default.md`](references/arkcli-profile-set-default.md)
- Maintainer regression cases:
  [`references/evals.md`](references/evals.md)

Never change product, account, Region, or tenant as a fallback for a missing
profile or resource.
