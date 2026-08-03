# BytePlus account-readiness scope

Use this reference for real-name verification, balance, IAM, VMP, TOS, or
general account readiness.

## Commands

```bash
arkcli doctor account --format json
```

The diagnosis is read-only, does not register `--dry-run`, and each online
check degrades independently.

## Output contract

BytePlus uses only fields already present in the Volc `doctor account` schema.
BytePlus APIs can return additional data, but doctor must not expose
BytePlus-only fields.

| Output | BytePlus source | Interpretation |
|---|---|---|
| `identity` | Active local profile and identity | Current account, IAM user, root status, and auth method |
| `compliance.realname.verified` | `GetVerifyInfo.IsVerified` | Whether the BytePlus account has passed real-name verification |
| `compliance.realname.reason` | `GetVerifyInfo` probe | Why real-name verification data is unavailable |
| `compliance.balance.available` | `QueryBalanceAcct.AvailableBalance` | Current available account balance, preserved as a string |
| `compliance.balance.currency` | `QueryBalanceAcct.Currency` | Currency returned by the balance service |
| `compliance.balance.reason` | Balance probe | Why balance data is unavailable |
| `permissions.iam_system_policies` | IAM attached-policy lookup | Shared doctor IAM-policy evidence for the current identity |
| `ecosystem.vmp` | VMP subscription, service-linked role, and workspace binding | ModelArk monitoring readiness |
| `ecosystem.tos` | TOS account status | Object-storage readiness for dependent workflows |

Do not expect or invent these removed BytePlus-only fields:

- raw account-opening, payment-qualification, or payment-method fields;
- cash, frozen, arrears, credit-limit, or overdue balance fields;
- effective ModelArk-read IAM fields;
- `ecosystem.model_ark`;
- billing-overview data.

## Analysis order

1. Read `identity.ok`, `identity.is_root`, and `identity.reason`. Signed-out
   execution is supported; online checks degrade independently.
2. Read `compliance.realname.verified` and `reason` directly as the
   `GetVerifyInfo` real-name-verification result. Do not reconstruct it from
   account-opening or payment-qualification statuses.
3. Read balance strings exactly as returned. Do not parse through floating
   point, invent missing amounts, or infer overdue state.
4. Read IAM policy evidence. A root account short-circuits the attached-policy
   lookup. For an IAM user, custom policies can make name matching incomplete.
5. Read `ecosystem.vmp.type` before suggesting remediation, then inspect TOS
   independently.

## IAM interpretation

- A root account is authorized without calling the attached-policy API.
- For an IAM user, an attached Ark system policy is positive evidence.
- A custom policy can grant sufficient access even when no system-policy name
  matches. Treat `total_attached_count > 0` plus no match as inconclusive, not
  definitive denial.
- Use least privilege. `ArkReadOnlyAccess` is suitable for read-only diagnosis;
  broader operations may require `ArkFullAccess` or an equivalent custom
  policy.

Official ModelArk access-control reference:
https://docs.byteplus.com/en/docs/ModelArk/1263493

## Unknown and degradation rules

- A nonempty `reason` means one probe degraded. It does not invalidate
  successful sibling probes.
- Missing optional values are unknown. They are not zero or false.
- `authorized=false` based only on system-policy names does not prove a custom
  policy is ineffective.
- `verified=true` means `GetVerifyInfo.IsVerified` returned true.

## VMP remediation

- `vmp_not_open`: enable VMP and accept required terms in the console.
- `vmp_slr_missing`: grant the ModelArk service-linked role in BytePlus IAM.
- `vmp_workspace_unbound`: with approval, rerun the relevant model, Endpoint,
  or metrics command with `--auto-bind`.
- `ok=true`: monitoring dependencies are ready.

Do not use `doctor report` as a recovery path. It is unavailable on BytePlus.

## Boundaries

- This command is read-only and has no `--auto-bind` flag.
- It does not activate a model, enable VMP, create the service-linked role,
  change IAM, or make a payment.
- It does not invoke a model or consume inference tokens.
- Use only BytePlus Billing, IAM, VMP, TOS, and ModelArk semantics and URLs.
