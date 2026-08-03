# BytePlus ModelArk and common error codes

Use:

```bash
arkcli doctor error <code> --format json
```

The command returns `category`, `subtype`, `root_cause`, `hint`, and the
original `code`. If a model name or Endpoint ID is available, diagnose that
resource first and use this reference to interpret its error distribution.

The lookup is local, read-only, and does not require authentication. Preserve
the exact case and suffix from the upstream response. Authentication-looking
codes and error JSON still take this route when no model or Endpoint identifier
is present. Execute the lookup first and return its structured result or its
explicit unknown-code failure; do not replace the lookup with auth status,
profile inspection, login, or API-key repair unless the user separately asks
for credential remediation.

## Routing and output contract

| Available context | First command |
|---|---|
| Endpoint ID plus error code | `arkcli doctor infer-endpoint <endpoint-id> --format json` |
| Foundation-model name plus error code | `arkcli doctor model <model-name> --format json` |
| Error code only | `arkcli doctor error <code> --format json` |
| RequestId only | Preserve it and ask for the resource and error code; doctor has no RequestId reverse lookup |

`doctor error` returns:

- `code`: the exact input code;
- `category`: ModelArk/Ark or common BytePlus gateway family;
- `subtype`: stable routing family;
- `root_cause` and `hint`: BytePlus-specific interpretation and next step;
- `rules`: disambiguation signals when applicable;
- `needs_backend`: capabilities that are not locally executable;
- `skill=arkcli-doctor` and `reference=error-codes`.

Unknown codes fail explicitly. Do not replace an unknown code with another
product's similar code. In particular, `ModelAccessDenied` is not a registered BytePlus
doctor code. Use the exact BytePlus response, such as `ModelNotOpen`,
`AccessDenied`, or `OperationDenied.PermissionDenied`.

For a resource-first diagnosis, match the leading code in `errors.overall` or
`errors.task` to the closest section below. Do not discard the model/Endpoint
window, count, percentage, or quota evidence after looking up the code.

## Official BytePlus common OpenAPI errors

Source:
https://docs.byteplus.com/en/docs/byteplus-platform/reference-common-error-codes

| Code | CodeN | HTTP | Meaning and action |
|---|---:|---:|---|
| `MissingParameter` | 100002 | 400 | Add the required parameter named by the message |
| `MissingAuthenticationToken` | 100003 | 401 | Add the required authentication token or signed headers |
| `MissingRequestInfo` | 100004 | 400 | Restore required request metadata such as `X-Date` |
| `MissingSignature` | 100005 | 401 | Sign the request with arkcli or a BytePlus SDK |
| `InvalidTimestamp` | 100006 | 400 | Synchronize the local clock and regenerate the signature |
| `ServiceNotFound` | 100007 | 404 | Check the signed service name |
| `InvalidActionOrVersion` | 100008 | 404 | Check the action and API version |
| `InvalidAccessKey` | 100009 | 401 | Refresh or replace the AccessKey |
| `SignatureDoesNotMatch` | 100010 | 401 | Check SecretKey, region, service, canonical query, and body hash |
| `LackPolicy` | 100012 | 403 | Attach the required BytePlus IAM policy |
| `AccessDenied` | 100013 | 403 | Check IAM and resource-level access |
| `InternalError` | 100014 | 500 | Retry with backoff and preserve `RequestId` |
| `FailToConnect` | 100015 | 502 | Retry; the common gateway could not reach the backend |
| `InternalServiceTimeout` | 100016 | 504 | Retry idempotently with backoff |
| `FlowLimitExceeded` | 100018 | 429 | Throttle and retry with backoff |
| `ServiceUnavailableTemp` | 100019 | 503 | Retry after a delay |
| `MethodNotAllowed` | 100020 | 405 | Use the documented HTTP method |
| `InternalServiceError` | 100023 | 502 | Retry and preserve `RequestId` |
| `InvalidAuthorization` | 100024 | 400 | Correct the Authorization header format |
| `InvalidCredential` | 100025 | 400 | Correct the Credential component |
| `InvalidSecretToken` | 100026 | 401 | Refresh the STS token |

BytePlus explicitly states that product-specific errors are documented by the
target product. The public table is therefore a gateway layer, not the complete
ModelArk catalog.

The standard response envelope places errors under
`ResponseMetadata.Error` and exposes `RequestId` in `ResponseMetadata`.
Preserve that identifier when escalating:
https://docs.byteplus.com/en/docs/byteplus-platform/reference-request-response

## ModelArk access errors

| Code or family | Likely cause | Action |
|---|---|---|
| `ModelNotOpen` | Model access is not enabled | Enable the model in ModelArk |
| `AccessDenied` | IAM or resource permission is missing | Check Ark policies and active profile |
| `OperationDenied.PermissionDenied` | Foundation-model configuration is not visible | Grant the required ModelArk permissions |
| `OperationDenied.ServiceNotOpen` | Model or dependency is not enabled | Enable the named service in BytePlus |
| `InvalidEndpointOrModel.NotFound` | Wrong ID, region, or visibility | Check the active profile, region, and exact resource ID |
| `InvalidEndpointOrModel.ModelIDAccessDisabled` | Direct model-ID invocation is disabled | Invoke through an inference Endpoint |

Official ModelArk access-control reference:
https://docs.byteplus.com/en/docs/ModelArk/1263493

## Rate limits and burst protection

Use `doctor infer-endpoint` or `doctor model` to distinguish Endpoint pressure
from model or account pressure.

Important families include:

- `RateLimitExceeded.EndpointRPMExceeded`
- `RateLimitExceeded.EndpointTPMExceeded`
- `ModelAccountRpmRateLimitExceeded`
- `ModelAccountTpmRateLimitExceeded`
- `ModelAccountIpmRateLimitExceeded`
- `APIAccountRpmRateLimitExceeded`
- `AccountRateLimitExceeded`
- `QuotaExceeded`
- `ServerOverloaded`
- `RequestBurstTooFast`

For `ServerOverloaded` and `RequestBurstTooFast`, ramp traffic gradually and
retry with exponential backoff. BytePlus documents the burst behavior here:
https://docs.byteplus.com/en/docs/modelark/handle_burst_traffic

Do not retry validation, authentication, or permission errors blindly.

## Account and billing errors

- `AccountOverdueError` or `OperationDenied.ServiceOverdue`: inspect outstanding
  bills and payment methods in BytePlus Billing Center.
- `InvalidSubscription`: inspect the named product-plan subscription.

Billing references:

- Overview: https://docs.byteplus.com/en/docs/byteplus-platform/docs-getting-started
- Paying bills: https://docs.byteplus.com/en/docs/byteplus-platform/docs-paying-bills

## Validation and resource errors

`MissingParameter.<field>`, `InvalidParameter.<field>`, and
`NotFound.<resource>` use their base diagnosis while preserving the full code.
Read the message for the exact field or resource name.

Do not remove parameters until the target ModelArk API confirms they are
unsupported. Do not change region or tenant merely to make a resource appear.

## Content safety

The command groups content-safety codes into stable subtypes:

- `input_real_face`
- `input_copyright`
- `input_content_safety`
- `output_video_copyright`
- `output_video_safety`
- `sensitive_content`
- `risk_detection`

Replace or revise the input with compliant, authorized material. Never claim
that a local transformation guarantees policy acceptance. `needs_backend`
lists unavailable automated remediation and is not an executable feature.

BytePlus does not support `doctor report`. Do not suggest it for generated
content, model, or Endpoint errors.
