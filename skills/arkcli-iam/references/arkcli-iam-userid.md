# iam userid

Resolve BytePlus IAM UserIDs without enumerating the full account directory.

## Commands

```bash
# Resolve one username prefix
arkcli iam userid --username <prefix> --format json

# Resolve multiple prefixes while preserving input order
arkcli iam userid --username <prefix-1>,<prefix-2> --format json

# Return the current logged-in identity
arkcli iam userid --format json
```

## Parameters

| Parameter | Required | Description |
|---|---|---|
| `--username` | No | Comma-separated username prefixes. Omit only when the user wants the current identity UserID. |
| `--format json` | Recommended | Keep `queries[].matches[]` machine-readable for downstream commands. |

## Output contract

```json
{
  "queries": [
    {
      "query": "alice",
      "matches": [
        {"user_id": "123456", "user_name": "alice"}
      ]
    }
  ]
}
```

- Each input prefix keeps one `queries` entry, including zero-match results.
- `matches` contains only usernames with the requested prefix.
- `truncated: true` means the bounded server result may contain more matches. Ask for a more specific prefix instead of widening to an account-wide scan.
- When `--username` is omitted, the response uses one entry with `query: ""` and the current identity in `matches`.

## Safety and fallback rules

- This product command sends each non-empty prefix as a bounded IAM query and then applies strict username-prefix filtering.
- Stop and return the original error if authentication, permission, timeout, or transport fails.
- Never retry with `arkcli api iam.list_users`, an empty query, pagination over the full directory, or local filtering of all users.
- Do not guess a UserID. If there are multiple matches, show them and ask the user to select the exact identity before any seat assignment.

## Downstream seat assignment

After the user selects an exact match, pass its `user_id` to the plan command documented by [`arkcli-plans`](../../arkcli-plans/SKILL.md). Resolving identity does not authorize a seat mutation; the downstream write still requires its own confirmation policy.
