# BytePlus CLI health scope

Use this reference for `arkcli doctor` with no subcommand. It is the first
diagnostic path for installation, DNS, TCP/TLS, clock, profile, or credential
health.

## Commands

```bash
arkcli doctor --format json
```

This scope is safe while signed out and does not register `--dry-run`. Execute
it as the first and direct command.
Do not run `arkcli auth status`, `arkcli profile list`, `arkcli profile show`,
or configuration initialization first when the user's goal is to diagnose why
the CLI or credentials do not work; the doctor output already carries the
relevant local authentication and configuration state.

## Checks and schema

| Output | Meaning |
|---|---|
| `installation.cli_version` | Version injected into the current binary |
| `installation.latest_version.available` | Whether the latest BytePlus npm package version could be resolved |
| `installation.latest_version.up_to_date` | Version comparison when both versions are comparable |
| `installation.latest_version.reason` | Registry lookup or development-build limitation |
| `installation.path` | Absolute executable path |
| `connectivity.endpoint` | Resolved BytePlus BaseURL used for the probe; when no profile or explicit override is configured, this is the BytePlus product default |
| `connectivity.dns.ok` | DNS resolution result |
| `connectivity.dns.addresses` | Resolved addresses when available |
| `connectivity.tcp.ok` | TCP and TLS handshake result |
| `connectivity.tcp.tls_version` | Negotiated TLS version |
| `connectivity.clock_skew.ok` | Whether observed clock offset is within the threshold |
| `connectivity.clock_skew.available` | Whether a server `Date` header was available |
| `connectivity.clock_skew.offset_ms` | Signed local/server time offset |
| `connectivity.clock_skew.direction` | `local_ahead`, `local_behind`, or `in_sync` |
| `connectivity.clock_skew.threshold_ms` | Current warning threshold |
| `auth.logged_in` | Whether a usable local identity was resolved |
| `auth.auth_method` | `sso`, `aksk`, `apikey`, `sts`, or `none` |
| `auth.credential_state` | `active`, `expired`, `absent`, or `unknown` |
| `configuration.profile` | Active profile name |
| `configuration.profile_expired` | Whether the selected profile has expired |
| `configuration.api_key.status` | API Key verification state when that check is available |
| `configuration.verified` | Overall configuration verification result |
| `configuration.reason` | Structured degradation reason |

The package lookup is a version check, not a product API call. DNS, TCP/TLS,
and clock checks use the active BytePlus BaseURL. This scope does not call a
ModelArk model and does not consume inference tokens.

## Analysis order

1. Verify `installation.path` and `installation.cli_version` identify the
   intended BytePlus build. Do not infer the product from the filename alone.
2. Read `connectivity.dns.ok`. If false, later TCP and clock checks can be
   skipped; use each check's `reason` instead of inventing another failure.
3. Read `connectivity.tcp.ok` and `tls_version`. A reachable TLS endpoint is a
   connectivity success even when a later signed API request would return 401
   or 403.
4. Read `clock_skew.available` before `clock_skew.ok`. An unavailable server
   time is unknown, not a zero offset.
5. Read `auth.credential_state` and `configuration.reason`. Route missing or
   expired credentials to `arkcli-auth`, then resume the original task.
6. Continue to account, model, Endpoint, or metrics diagnosis only after the
   local root cause is resolved or excluded.

## Finding to action

| Finding | Meaning | Next action |
|---|---|---|
| `dns.ok=false` | The active BytePlus host cannot be resolved | Check local DNS, proxy, VPN, and the active BytePlus profile |
| `tcp.ok=false` after DNS succeeds | Route, firewall, proxy, TLS, or certificate failure | Repair the current network path; do not change tenant or Region |
| `clock_skew.available=true` and `clock_skew.ok=false` | Signed OpenAPI calls can fail | Synchronize the operating-system clock and rerun doctor |
| `credential_state=absent` | No usable BytePlus credential | Run `arkcli auth login`, then resume the task |
| `credential_state=expired` | Existing identity has expired | Run `arkcli auth login`, then resume the task |
| `credential_state=unknown` | Configuration could not be loaded or classified | Read `auth.reason` and `configuration.reason`; do not claim signed-out state without evidence |
| `configuration.api_key.status` is disabled or missing | The selected API Key is unusable | Follow the BytePlus auth workflow to refresh or reselect it |
| `latest_version.available=false` | Version lookup failed or is not meaningful | Read `reason`; do not label the installed CLI broken if all other checks pass |

## Boundaries

- The control-plane Region remains `ap-southeast-1`.
- Do not replace the active BaseURL with another product's host.
- Do not expose resolved credentials or signed headers.
- Do not turn a skipped or unavailable check into `ok=true`.
- If connectivity and credentials are healthy, stop this scope and continue
  with the user's actual account, model, Endpoint, or metrics goal.
