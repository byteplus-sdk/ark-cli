# arkcli-models evals

## Coverage goals

- Model search requests should use `arkcli models search`; avoid using `list` for capability recommendations.
- Model asset inventory requests should use `arkcli models list --page-all`; avoid jumping to Raw API Explorer.
- "Custom models from the past N days" should be filtered in local JSON by `create_time` and custom-model fields, without asking the user to use the console.

## Trigger / should trigger

- Trigger `arkcli-models` when the user explicitly asks for a model list, model details, model search, or model statistics.
- Trigger the `models list` path when the user asks for asset inventory such as "mine/custom/recently created".

## Anti-trigger / should not trigger

- When the user wants to chat, generate, or deploy directly, do not treat `arkcli-models` as the destination. Enter it briefly only when the model name is missing.
- When the user explicitly wants a Raw API Action, OpenAPI list, or low-level params, do not use this skill; route to `arkcli-api-explorer`.

## Guard

- When authentication fails, a 401 occurs, or login state is unknown, first check `arkcli auth status`.
- When profile, region, or base URL is abnormal, route to `arkcli-config` for configuration troubleshooting.
- A missing server-side time-filter flag is not a failure condition. First filter local JSON and explain the limitation only after confirming the request cannot be completed.

## Tested happy-path CLI commands

```bash
arkcli models list --page-all --sort-by CreateTime --sort-order Desc --format json
arkcli models search --modality text --min-context-window 200000 --capability thinking --strict-filter
```

## Regression cases

| case | prompt | Expected |
|------|--------|------|
| `uat-models-list-recent-custom-001` | How many custom models did I create in BytePlus in the past seven days? List them. | Use `arkcli models list --page-all --format json`, and locally filter `create_time` and Custom fields; forbid `arkcli api --list` and do not suggest using the console |
| `models-list-owned-custom` | Help me see which custom models I have, and output the count and model names. | Use `arkcli models list` for asset enumeration, then perform client-side filtering and statistics |
| `models-search-task-fit` | Help me find a text model that supports thinking and has more than 200K context for +chat. | Use `arkcli models search --min-context-window ... --capability thinking` |
| `models-search-speech-boundary` | Does the BytePlus ModelArk marketplace have TTS models? Can I deploy one directly? | Use `arkcli models search <tts/voice keyword>` for marketplace discovery; clearly state that voice models currently do not support `+deploy` / usage / pricing / onboard; do not recommend `+chat` or `+gen` |

## Key scoring points

- Must route to `arkcli-models`.
- Asset inventory must recommend `arkcli models list`, preferably with `--page-all` and `--format json`.
- Time filtering must explain client-side filtering or local JSON processing.
- Must not recommend `arkcli api --list` as the preferred path.
- Must not claim that the CLI lacks the capability or send the user to the console merely because a server-side time flag is missing.
- After a voice-model match, must stop at the marketplace discovery layer and must not continue recommending deployment, usage, or pricing queries.
