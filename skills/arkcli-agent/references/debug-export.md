# Debug / Export / Validate

## Debug / Export

- `+debug <session-id>` Aggregates session get, recent events, resources, threads, outputs status, error events, pending actions, and warnings. Use `--limit` to control the number of events, and `--session-thread-id` to focus on a single thread.
- `+export <session-id>` Exports to `arkcli-session-<session-id>-<timestamp>.tar.gz` with `--output` to specify the path. The archive includes `manifest.json`, `session.json`, `events.json`, `resources.json`, `threads.json`, and `notes.md`. If the workspace tarball and memory snapshot are not available for reading, the manifest is marked as unsupported without fabricating content.
- Before writing the archive, run the same `+export <session-id> ... --dry-run --format json` command and inspect the four read steps, archive entries, and local output path in the zero-network `preview.v1` plan. Without `--output`, the timestamp in the default filename is an `unresolved` placeholder. Preview neither reads the session nor creates or replaces the archive.

## End-to-End Validation Template

Run every line below as a **separate shell or tool call**. Pass an ID to the next step only after the preceding step succeeds. If a step times out, diagnose the known ID separately.

```bash
arkcli agent agent get <agent-id> --format json
arkcli agent env create --name arkcli-<domain>-env-<timestamp> --config '{Type: cloud, Networking: {Type: unrestricted}}' --format json
arkcli agent session create --agent-id <agent-id> --environment-id <env-id> --title arkcli-<domain>-session-<timestamp> --format json
arkcli agent session events send <session-id> --type user.message --text "<one small test task>" --format json
arkcli agent session events stream <session-id>
arkcli agent session get <session-id> --format json
arkcli agent session events list <session-id> --limit 20 --format json
arkcli +tail <session-id> --session-thread-id <thread-id>
arkcli agent session threads list <session-id> --limit 10 --format json
arkcli agent session resources list <session-id> --format json
```

When the user requests to keep resources, retain the created agent/env/session/vault/credential.
