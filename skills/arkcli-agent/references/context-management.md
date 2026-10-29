# Automatic context compaction configuration

`ContextManagement` configures automatic compaction; it does not immediately compact a session.
Use `agent session compact` for an immediate request. Do not add configuration unless requested.
Backend acceptance and rollout must be verified from actual responses, not from CLI flag availability.

The array currently accepts zero or one policy:

| Field | Meaning |
| --- | --- |
| `Type` | Required; `compaction` |
| `CompactThreshold` | Optional integer, full-context token threshold that triggers compaction, not the output length. The backend requires at least 50000 and, for Ark models with known context windows, at most floor(window × 0.8). BYOK has no context-window upper-bound check |
| `Instructions` | Optional additional compaction instructions; omitted or empty uses the built-in prompt. The backend allows at most 100000 characters and rejects non-empty whitespace-only text; do not impose the frontend input's 400-character limit |

Omission/null on create leaves defaults to the backend. On update/override it keeps existing/inherited configuration.
`[]` clears custom settings, **not automatic compaction**. A non-empty array replaces the whole policy list;
`[{"Type":"compaction"}]` restores a policy without a custom threshold or prompt. Missing fields do not merge with the old policy.
Public responses may fill display defaults (currently dynamic configuration with a 192000 fallback).
Never hardcode that value into requests or treat displayed defaults as explicit user settings.

```bash
arkcli agent agent create --name my-agent --model <model-id> \
  --context-management '[{"Type":"compaction","CompactThreshold":64000,"Instructions":"Retain goals and pending work"}]'
arkcli agent agent update <agent-id> --context-management '[]'
arkcli agent session create --agent-id <agent-id> --environment-id <env-id> \
  --agent-overrides '{"ContextManagement":[{"Type":"compaction","CompactThreshold":64000}]}'
```

Resolve real IDs before execution. Agent flags accept a JSON/YAML array or null, including via `@file`; complete `--file`/stdin requests
accept `ContextManagement`. Explicit flags override file values, including null: it does not fall back to the file policy;
create uses defaults and update retains the server policy. Only `[]` clears custom settings. An empty string is not null and is rejected. PascalCase, camelCase and snake_case keys are accepted;
prompt contents are preserved. `+new-agent` and `+iterate` accept the same flag. Fork preserves source configuration;
use an explicit `[]` to clear it. With `+iterate`, this flag updates the Agent itself.

For a new Session use `--agent-overrides` (also accepts `@file`) or `AgentWithOverrides.ContextManagement` in its request,
not a top-level Session field. This affects only the new Session, not the source Agent or existing sessions.
Use `{"ContextManagement":[]}` to clear inherited custom configuration. Business validation belongs to the server;
do not fetch model metadata as a write gate or silently clamp values.

Read back with `agent agent get` or `agent session get --format json`. A stored configuration does not prove that
compaction ran; inspect events separately after context reaches the threshold. `--dry-run` verifies local construction,
not backend deployment or parameter acceptance.
