# BytePlus Account-Opening and Payment Verification Gate

> Read this reference before a command activates a model or creates, deploys,
> fine-tunes, or provisions a billable resource.

BytePlus account-opening and payment verification form a required product gate.
The public CLI contract is:

```text
byteplus_sso.identity.verified
```

Ark CLI derives that boolean from two independent checks:

| Check | Complete | Incomplete |
|---|---|---|
| Account opening: `GetProfileInfo.Result.Status` | `1` or `2` | `0` |
| Payment: `CheckPaymentQualification.Result.QualificationStatus` with fixed `SubjectNo=1065` | `2` | `1` |

`verified` is `true` only when both checks are complete. If either check is
known to be incomplete, `verified` is `false`. If a probe fails or returns an
unknown status and neither check is known to be incomplete, Ark CLI omits the
field instead of guessing.

## Mandatory Preflight

For model activation, deployment, inference Endpoint creation, fine-tuning
creation, and Managed Agent creation:

1. For a workflow that can automatically activate a model or is about to
   create, deploy, fine-tune, or provision a billable resource, run:

   ```bash
   arkcli auth status --format json
   ```

   Run it before the first model-availability or activation check that can lead
   to the write. Do not generalize it into a gate for unrelated read-only
   inspection.

2. Read `byteplus_sso.identity.verified`:
   - `true`: continue the original workflow.
   - `false`: stop and send the user to
     `https://console.byteplus.com/user/basics/`. Wait for the user to complete
     both account-opening and payment verification before retrying.
   - missing: treat the result as unknown. Continue normally and preserve any
     structured backend error and request ID if the write fails.

The public `verified` field is a combined result. When it is `false`, do not
claim whether account opening, payment verification, or both are incomplete.
Both incomplete states use the same BytePlus account settings URL.

Do not replace this check with raw `GetProfileInfo` or
`CheckPaymentQualification` API calls. The CLI owns the fixed subject number,
status mapping, product isolation, and error-degradation behavior.

## Failure Handling

- `account_not_verified`: stop, preserve this error type, and use
  `https://console.byteplus.com/user/basics/`.
- Do not retry with another model, account, Region, project, or product.
- Do not poll the verification APIs while waiting for the user.
- Do not treat successful SSO, an active API key, or one completed check as
  proof that the full gate passed.
- Do not report a missing `verified` field as either verified or unverified.

The shared write gate performs the same check before automatic model
activation. Backend `NotVerifiedAccount` errors are also translated to the
same `account_not_verified` type and BytePlus account settings link.
