# Todo Write planning tool

Read this when the user explicitly requests a task-plan/todo tool. `todo_write` is built in and uses Agent create/update `--tool`; it is not a custom tool or a separate Todo CLI command. Product rollout and access remain subject to the BytePlus service; Volc availability is not proof of BytePlus availability.

Use configuration name `todo_write` within `agent_toolset_multimodal_20260825`, never a base toolset such as `agent_toolset_20260701`. It needs no `Models` or tool-model discovery. Do not enable image/video tools just because the toolset name contains multimodal. Stop on permission or unsupported-toolset errors, preserve the error and RequestId, and do not switch toolsets or use Raw API to bypass the rejection.

This is **one toolset entry, not the complete Tools array**. When adding this toolset for Todo only, explicitly disable its defaults to avoid enabling unrelated tools or triggering their model dependencies:

```json
{
  "Type": "agent_toolset_multimodal_20260825",
  "DefaultConfig": {
    "Enabled": false,
    "PermissionPolicy": {"Type": "always_allow"}
  },
  "Configs": [
    {
      "Name": "todo_write",
      "Enabled": true,
      "PermissionPolicy": {"Type": "always_allow"}
    }
  ]
}
```

- Creation: explicit `--tool` replaces all default tools. Combine this entry with the complete base-tool configuration the user needs; passing this entry alone drops other tools.
- Update: first run `arkcli agent agent get <agent-id> --format json`. If the toolset already exists, merge the Todo configuration into it without adding a duplicate or overwriting its DefaultConfig with this example. Preserve other tools, model bindings, and PermissionPolicy, even if the frontend hides permission controls.
- Submit with `arkcli agent agent update <agent-id> --tool '<complete Tools JSON array>' --format json`, then read back. To disable Todo, set its Enabled to false while retaining other settings.
- For runtime verification, create a Session bound to the updated Agent version; do not assume an existing Session inherits the update. Ask it to use Todo to maintain a plan and inspect tool events. An accepted configuration alone is not proof of a tool invocation.

Runtime events may use `todo_write` or `TodoWrite` depending on BaseAgent; configuration always uses `todo_write`. Agent Loop executes it internally, so no custom-tool implementation or manually returned custom-tool result is needed.
