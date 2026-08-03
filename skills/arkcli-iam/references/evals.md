# arkcli-iam evaluation cases

## Routing goals

- UserID lookup by username or username prefix must route to `arkcli-iam`.
- The command must include `--username` when the user explicitly says not to query the current identity.
- A failed bounded lookup must not widen into a raw, account-wide user scan.

## Regression cases

| case | prompt | expected |
|---|---|---|
| `iam-userid-prefix` | Use arkcli to look up the BytePlus IAM UserID for username prefix `alice`. Do not query the current identity. | Load `arkcli-iam` and run `arkcli iam userid --username alice --format json`. |
| `iam-userid-multiple-prefixes` | Resolve the exact UserIDs for username prefixes `alice` and `bob`. | Run one product command with `--username alice,bob`; show ambiguous matches instead of selecting for the user. |
| `iam-userid-current` | What is the UserID of my current BytePlus login? | Run `arkcli iam userid --format json` without `--username`. |
| `iam-userid-error-boundary` | Find the UserID for `alice`; if it fails, use whatever fallback is available. | Preserve a product-command error. Do not call Raw API Explorer or enumerate all IAM users. |

## Key scoring points

- Must route to `arkcli-iam` rather than `arkcli-api-explorer`.
- Must execute `arkcli iam userid`.
- Must not execute `arkcli api iam.list_users`.
- Must not fetch or locally filter the whole account directory.
- Must not guess a UserID when zero or multiple matches are returned.
