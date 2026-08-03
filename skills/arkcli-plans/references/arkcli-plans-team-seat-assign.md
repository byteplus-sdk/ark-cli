# plans team seat-assign

> **Prerequisite:** Read [`../../arkcli-shared/SKILL.md`](../../arkcli-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

> **Write operation:** Modifies bindings between enterprise seats and sub-users. One `GrantSeats` call supports batches; the server returns per-item status (success and failure can coexist).

Bind Coding Plan enterprise-edition seats to IAM sub-users. The same Action supports the enterprise edition (`coding-plan-team`); the server infers the plan line from SeatID, so **Scene is not required**.

## What if the employee's exact UserID is unknown

If you know only the employee username (or prefix), first resolve it with `arkcli iam userid`:

```bash
# One prefix
arkcli iam userid --username ivan

# Query multiple employees at once
arkcli iam userid --username ivan,bob,carol
```

Output format:
```json
{
  "queries": [
    {
      "query": "ivan",
      "matches": [
        {"user_id": "12345", "user_name": "ivan"},
        {"user_id": "67890", "user_name": "ivanka"}
      ]
    }
  ]
}
```

- Strict **prefix matching**: `iv` does not match `kevin`; it matches only `ivan` / `ivanka`.
- All results are returned when one prefix has multiple matches, allowing you to choose.
- A query record is retained for zero matches, making script decisions easier.
- Omitting `--username` returns the current identity's UserID ("who am I").

Pass the resulting `user_id` to `seat-assign --bind`. This skips the IAM lookup below and saves one round trip.

## Commands

```bash
# Bind one seat (automatically calls IAM ListUsers to resolve UserName)
arkcli plans team seat-assign --plan coding-plan-team \
    --bind seat-001=83144215

# Batch binding (repeat --bind)
arkcli plans team seat-assign --plan coding-plan-team \
    --bind seat-001=83144215 \
    --bind seat-002=83143634

# Explicitly specify UserName and skip IAM lookup (escape hatch when IAM permission is missing or to save a call)
arkcli plans team seat-assign --plan coding-plan-team \
    --bind seat-001=83144215:ivan \
    --bind seat-002=83143634:bob

# Custom project
arkcli plans team seat-assign --plan coding-plan-team \
    --bind seat-001=83144215 \
    --project-name my-project
```

## Parameters

| Parameter | Required | Type | Description |
|------|------|------|------|
| `--plan` | Yes | string | `coding-plan-team` |
| `--bind` | Yes | stringArray | Repeat for batch operations. Format: `seat-id=user-id` or `seat-id=user-id:user-name` |
| `--project-name` | No | string | Project name, default `default` |

> **`--bind`, not `--seat-id=...`:** The user's original spec used `--<seat-id>=<user-id>`, a dynamic flag name unsupported by cobra/pflag. It was changed to a repeatable fixed `--bind` flag with a value.

### Three-part syntax / IAM lookup

`--bind` supports two forms:

| Form | Parsing | Behavior |
|---|---|---|
| `seat-001=83144215` | UserName left empty | The service layer automatically calls **BytePlus IAM `ListUsers`**, paginates through all sub-users in the account, and matches UserName by ID |
| `seat-001=83144215:ivan` | UserName explicit | Skips IAM lookup and passes the value directly to `GrantSeats` |

The server's `GrantSeats` strictly validates all three fields `{SeatID, UserID, UserName}` (**UserName must be non-empty**), so the CLI must first obtain UserName. IAM lookup is the default path. If STS credentials lack `iam:ListUsers` permission or you want to avoid the IAM round trip, use the `:user-name` escape hatch.

IAM lookup performs **one list-all + in-memory map cache**. Multiple bindings in the same seat-assign call reuse the same cache (not N round trips per binding).

## Return value

> **Sensitive output:** `success[].api_key` is a one-time plaintext API key.
> Never quote, paste, persist, or echo its value in chat or logs. Tell the user
> only that the command returned a key and that they must save the local stdout
> securely. The `"sk-..."` value below is a redacted placeholder, not a real
> key.

```json
{
  "plan": "coding-plan-team",
  "project_name": "default",
  "success_count": 2,
  "failed_count": 0,
  "success": [
    {
      "seat_id": "seat-001",
      "user_id": "83144215",
      "api_key_sid": "apikey-...",
      "api_key": "sk-..."
    }
  ],
  "failed": []
}
```

| Field | Description |
|------|------|
| `success[].api_key` | Plaintext APIKey after binding this seat; tell the user to save it immediately |
| `success[].api_key_sid` | Stable APIKey ID used to locate the resource for later rotation / deletion |
| `failed[].reason` | Server `FailedReason` passed through, typically `BindCountLimitExceeded` |

## Failure semantics

**Partial failure** is a valid terminal state:
- stdout always contains the complete result (both success + failed arrays).
- When `failed_count > 0`, stderr adds a `grant_seats_partial` envelope and exit code = 5 (`ExitAPI`).
- When `failed_count == 0`, exit code = 0.

| Error | Cause | Handling |
|------|------|------|
| `--plan must be one of ...` | Misspelled or personal plan used | Use `coding-plan-team` |
| `--bind ... must be in the form seat-id=user-id` | Invalid format | Strictly use `seat-id=user-id` or `seat-id=user-id:user-name` |
| `--bind specifies SeatID %q more than once` | Same SeatID appears in multiple `--bind` flags | Server rejects it; client blocks first |
| `resolve UserName for UserID=...: IAM ListUsers ...: AccessDenied` | STS credentials lack IAM permission | Use the escape hatch: explicitly specify `--bind seat-id=user-id:user-name` |
| `failed_count > 0` + `BindCountLimitExceeded` | The sub-user already has N seats bound (limit reached) | Unbind first (unbind subcommand not yet implemented); frontend uses `RevokeSeats` |

## Notes

- IAM lookup is a **full listing** (paginated retrieval of all IAM users in the account), triggered only by the first binding; subsequent lookups in the same call use memory.
- Defensive IAM lookup fallback: if the target ID is still not found after more than 10,000 entries, report an error (prevents endless pagination caused by a backend cursor loop).
- One seat **cannot simultaneously** bind multiple sub-users (server constraint). To change the user, unbind first, then assign (unbind = `RevokeSeats`, not yet exposed by the CLI).
- `success[].api_key` is the **newly generated plaintext APIKey**, effective immediately after binding. After the first response, neither console nor CLI can retrieve the original plaintext (only masked form), so save it immediately.

## References

- [arkcli-plans](../SKILL.md) -- Skill overview
- [`plans team seat-list`](arkcli-plans-team-seat-list.md) -- List idle seats first
- [arkcli-shared](../../arkcli-shared/SKILL.md)
