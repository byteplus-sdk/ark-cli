# `auth status` and `auth logout`

## Inspect status

```bash
arkcli auth status --format json
```

Use status as the first authentication diagnostic. Expected categories include:

- `logged_in` and `auth_method`;
- top-level `identity_type`, with the canonical value `individual` or
  `enterprise`; this is an account property and does not belong to the SSO
  session;
- masked SSO state and identity claims under `byteplus_sso`;
- `byteplus_sso.identity.verified` when the two BytePlus verification probes
  produced a known result;
- top-level `control_plane_auth` for STS/control-plane readiness;
- masked `ark_api_key` metadata and remote status when available;
- effective top-level `project_name`;
- `active_profile`, including the selected profile's `region`;
- `profiles_summary` for the local profile inventory;
- a structured hint when the identity needs login or refresh.

Never infer a complete secret from masked output and never read local token
files to enrich the response.

Keep these facts separate:

- `logged_in=true` means a usable authentication method is active;
- `identity_type` classifies the current account principal independently of
  the login method;
- `control_plane_auth.status` describes whether control-plane calls are ready;
- `ark_api_key.status` describes the selected data-plane API key;
- `byteplus_sso.identity.verified` describes account-opening and payment
  verification.

A successful login or an active API key does not imply that account
verification is complete.

## Interpret account verification

`byteplus_sso.identity.verified` is the combined BytePlus account-opening and
payment verification result:

- `true`: account-opening status is `1` or `2`, and payment qualification
  status is `2`;
- `false`: at least one check is known to be incomplete;
- absent: the combined state is unknown because a probe failed or returned an
  unsupported status.

The public boolean is intentionally combined. When it is `false`, do not claim
which individual check is incomplete. Use the same recovery destination for
either case:

```text
https://console.byteplus.com/user/basics/
```

For activation, deployment, Endpoint creation, fine-tuning creation, or
Managed Agent creation, follow
[`realname-gate.md`](realname-gate.md). Do not read another product's SSO
section, infer the result from login success, or call the two raw APIs
separately.

## Runtime scope diagnosis

If the effective project or Region is unexpected:

1. Compare top-level `project_name` and `active_profile.region` with
   `arkcli profile show --format json`.
2. Check which profile won through `--profile`, `ARK_PROFILE`, and the
   persisted default profile.
3. Remember that `--project-name`, `--region`, `ARK_PROJECT_NAME`, and
   `ARK_REGION` are not runtime overrides; change the selected profile instead.
4. Remember that the BytePlus Region is fixed to `ap-southeast-1`; another
   explicit Region is invalid, not a fallback target.
5. Confirm that the command is using `~/.arkcli-bp/` rather than state from a
   different product.
6. If the selected API key belongs to another project, explicitly reselect a
   valid BytePlus key instead of editing state files.

## Interpret key status

Treat these states separately when present:

- `active`: the selected key is present and active;
- `disabled`: the key exists but cannot be used;
- `notfound`: the cached selection is absent from the remote inventory;
- `unknown`: the remote status could not be determined.

Do not treat `unknown` as `active`, and do not rotate or replace a key without
the user's explicit intent.

## Logout

```bash
arkcli auth logout
```

Logout clears local BytePlus authentication state. It is destructive and must
follow an explicit request. After it completes, verify with:

```bash
arkcli auth status --format json
```
