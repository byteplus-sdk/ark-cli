# +deploy detailed reference

> **Prerequisite:** Read [`../SKILL.md`](../SKILL.md) first. This file only adds flag details, JSON examples, and error codes that are not covered above.

## Key points agents must read (do not skip)

1. Subcommands are exhaustive: only `arkcli +deploy`.**There is no** `arkcli deploy ...` / `arkcli endpoint create` / `arkcli +deploy create`.
2. **Write operation + billing**: This workflow does not support `--dry-run`. The first invocation without `--yes` returns a `requires_confirmation` plan and creates nothing. After the user sees the normalized plan and gives a fresh explicit confirmation, rerun the same command with `--yes` to create for real.
3. JSON-type flag parameter names are always **PascalCase**: `Rpm`, `Tpm`, `Strategy`, and `Mode`, not lowercase.
4. When `--model cm-xxxxx` is executed for real, an existing Running endpoint is reused first. It is created only when no reusable endpoint is found.

## Command templates

```bash
# Stage 1: normalize and disclose the final plan; no model activation or Endpoint creation
arkcli +deploy --name my-endpoint --model dola-seed-2-1-turbo-260628

# Show the returned model/name/region/configuration/billing impact and stop.
# Only after a fresh explicit confirmation, stage 2 creates the disclosed Endpoint:
arkcli +deploy --name my-endpoint --model dola-seed-2-1-turbo-260628 --yes

# Stage 1 with rate limits (note the PascalCase parameter names)
arkcli +deploy --name my-endpoint --model dola-seed-2-1-turbo-260628 \
  --rate-limit '{"Rpm": 60, "Tpm": 10000}'

# With moderation
arkcli +deploy --name my-endpoint --model dola-seed-2-1-turbo-260628 \
  --moderation '{"Strategy": "Basic"}'

# With intelligent routing
arkcli +deploy --name my-endpoint --model dola-seed-2-1-turbo-260628 \
  --intelligent-router '{"Strategy": "Balanced", "Mode": "Automatic"}'

# Use a profile that owns the intended project
arkcli +deploy --name my-endpoint --model dola-seed-2-1-turbo-260628 \
  --profile platform_ap-southeast-1_my-project

# Custom model: reuse an existing Running endpoint directly if available; otherwise create one.
arkcli +deploy --name my-custom-endpoint --model cm-xxxxx
```

## Parameter

### Required

| Parameter | Type |Description|
|------|------|------|
| `--name` | string | Endpoint name |
| `--model` | string | Model ID (base model or custom model) |

### Common Optional

| Parameter | Type |Description|
|------|------|------|
| `--description` | string | Endpoint description |
| `--yes` | bool | Rerun the previously disclosed command after fresh confirmation to create the billable Endpoint. It does not authorize model activation in non-TTY environments. |
| `--rate-limit` | JSON string | Rate limit, such as `{"Rpm": 60, "Tpm": 10000, "Ipm": 100}` |
| `--moderation` | JSON string | Moderation configuration. Strategy: `Basic` / `Customized` / `Default` / `Skip` |
| `--view` | string | View endpoint details after creation. |

### Advanced configuration

| Parameter | Type |Description|
|------|------|------|
| `--model-unit-id` | string | Model unit ID |
| `--batch-only` | bool | Batch inference mode only. |
| `--data-delivery` | bool | Data delivery mode |
| `--domain` | string | Endpoint domain (automatically inferred from the model if not specified) |
| `--dedicated-to-ptu` | bool | Dedicated to PTU |
| `--dedicated-to-dynamic-ptu` | bool | Dedicated to dynamic PTU |
| `--content-generation` | JSON string | Content generation configuration |
| `--inference-foundry` | JSON string | Inference foundry configuration |
| `--tags` | JSON string | Tags, such as `[{"Key": "env", "Value": "prod"}]` |
| `--allow-data-collected` | bool | Allow data collection |
| `--service-info` | JSON string | Service information |
| `--need-watermark` | bool | Watermark required |
| `--watermark-info` | JSON string | Watermark information |
| `--intelligent-router` | JSON string | Intelligent routing. Strategy: `Balanced` / `CostFirst` / `EffectFirst`. Mode: `Automatic` / `CandidateSet` / `Ordered` |
| `--is-intelligent` | bool | Whether to enable intelligent routing |
| `--fallback` | JSON string | Fallback configuration |
| `--limit-coefficient` | JSON string | Limit coefficient |
| `--deployment-type` | string | Deployment type |
| `--is-agentic` | bool | Whether it is Agentic |
| `--agentic-strategy` | JSON string | Agentic strategy. Mode: `Auto` / `Custom` |
| `--is-aicc` | bool | Whether it is AICC |
| `--specify-region` | string | Specify region |

## Return values

The first invocation without `--yes` returns:

```json
{
  "status": "requires_confirmation",
  "operation": "create_endpoint",
  "model": "dola-seed-2-1-turbo-260628",
  "endpoint_name": "my-endpoint",
  "region": "ap-southeast-1",
  "configuration": {
    "description": "",
    "batch_only": false,
    "data_delivery": false,
    "dedicated_to_ptu": false,
    "dedicated_to_dynamic_ptu": false,
    "allow_data_collected": false,
    "need_watermark": false,
    "is_intelligent": false,
    "is_agentic": false,
    "is_aicc": false
  },
  "billing_impact": "...",
  "confirm_text": "...",
  "confirmation_required": true,
  "confirmation_flag": "--yes"
}
```

`configuration` always includes the normalized default fields shown above and adds every optional section supplied by the caller. Surface the CLI's actual complete `configuration`, not an empty or reconstructed placeholder.

Sensitive watermark input is the exception to literal echoing: `configuration.watermark_info` is a redacted summary. It reports `subject_name_present`, `social_unified_credit_code_present`, an optional `social_unified_credit_code_suffix`, and the non-sensitive dissemination setting without exposing the full subject name or credit code.

This stage performs no model activation and no Endpoint creation. Surface the exact payload and wait for a fresh explicit confirmation. Only then rerun the same command with `--yes`. A successful second-stage creation returns endpoint information.

## Common errors

| Error | Cause | Handling |
|------|------|------|
| Missing required parameter | `--name` or `--model` is not provided. | Must be specified. |
| JSON format error | JSON parameters such as `--rate-limit` have an incorrect format. | Wrap in single quotes and use PascalCase for parameters. |
| Model does not exist | `--model` ID is invalid. | Use `arkcli models search <keyword>` to find the correct name. |

## References

- [arkcli-deploy](../SKILL.md) -- skill entry
- [arkcli-shared](../../arkcli-shared/SKILL.md) -- shared authentication and global parameters
