## Zero Data Retention (ZDR) in Amazon Bedrock

**What it is:** ZDR is a data privacy configuration in Amazon Bedrock where no request or response data is written to durable storage by AWS or shared with the model provider. Once the response is returned, your data is gone from Bedrock's systems.

### Data retention modes

Amazon Bedrock controls data retention via a **mode** setting (not a simple on/off toggle):

| Mode | Behavior |
|------|----------|
| `none` | **Zero data retention.** No request or response data is written to durable storage by AWS or shared with the model provider. On the Responses API, `store` defaults to `false` and `store=true` is rejected. |
| `default` | The model's own retention policy applies. AWS may retain data for safety/abuse-prevention purposes. The model provider does not receive it. Previous ZDR models remain ZDR. |
| `provider_data_share` | Data may be retained and shared with model providers per their requirements. Required for certain models (e.g., Claude Fable 5, Claude Mythos 5). |
| `inherit` | Defer to a broader scope. This is the default for new accounts and projects. |

### How the effective mode is determined

Data retention is configured at two scopes, with the model's default as fallback:

```
effective mode = first non-inherit value of (project → account → model default)
```

- **Project** (most specific) — set via API
- **Account** — set via API
- **Model default** (least specific, read-only) — the model's built-in default

If your effective mode is not in a model's `allowed_modes`, the model appears as `unavailable` and requests are blocked.

### What ZDR (`mode: none`) guarantees:

- No request or response data is written to durable storage
- Data is **not shared** with model providers
- Data is **not used** to train or improve models
- On the Responses API, `store` defaults to `false`; `store=true` is rejected
- Background mode is not available
- Chat Completions and Messages requests are never retained

### What ZDR does NOT cover:

- **Your own logging** — if you enable CloudWatch Logs or S3 model invocation logging, that's your data in your account (your responsibility)
- **Model availability** — some models require `provider_data_share` and will be unavailable under `none` mode

### Important: `store=false` ≠ ZDR

Setting `store=false` on the Responses API does **not** guarantee zero data retention. Some models may still retain data for safety review even when `store=false` — data is retained but not retrievable by the customer. If you require **guaranteed** zero retention, set `data_retention_mode` to `none`.

### Why it matters:

| Concern | How ZDR addresses it |
|---------|---------------------|
| Regulatory compliance (HIPAA, GDPR, SOC 2, etc.) | No data at rest = reduced compliance scope |
| IP/trade secret protection | Proprietary prompts never stored by a third party |
| Model training concerns | Explicit guarantee your data won't be used to improve models |
| Data residency | No data persists outside the inference request lifecycle |

### ZDR vs. calling model providers directly:

| | Amazon Bedrock (`mode: none`) | Direct API (e.g., Anthropic, OpenAI) |
|--|-------------------------------|--------------------------------------|
| Data retention | None | Varies; may retain up to 30 days for trust & safety |
| Used for training | Never | Opt-out usually available, but policies vary |
| Data controller | You (AWS customer) | The model provider |
| Compliance scope | Your AWS account only | Extends to the provider's infrastructure |

### Models requiring data retention

Some models (e.g., Claude Fable 5, Claude Mythos 5) require `provider_data_share` mode. Under this mode:
- User prompts and completions are shared with Anthropic
- Data is retained for up to **30 days** for trust and safety purposes

If your organization requires ZDR for compliance but needs access to these models, contact your AWS account manager. ZDR access is evaluated on a **per-account, per-model basis** in coordination with the model provider.

### Enforcing ZDR with IAM / SCPs

You can enforce zero data retention across your organization using SCPs:

```json
{
    "Effect": "Deny",
    "Action": [
        "bedrock:PutAccountDataRetention"
    ],
    "Condition": {
        "StringNotEquals": {
            "bedrock:DataRetentionMode": "none"
        }
    }
}
```

This prevents anyone from setting data retention to anything other than `none`.

### Practical implications:

- ZDR is **not the default** — new accounts/projects default to `inherit`, which defers to the model's default
- You **should still enable** your own logging (CloudWatch/S3) if you need audit trails — ZDR means AWS won't retain them for you
- Under ZDR, model providers **never see your data** — Bedrock acts as a pass-through
- Some newer models may be **unavailable** under `mode: none` — check the model's `allowed_modes`

**In short:** ZDR (`data_retention_mode: none`) means Bedrock processes your request, returns the result, and retains nothing. Your data stays yours — but you trade access to certain models that require retention for safety purposes.

---

### References

- [Data retention in Amazon Bedrock – AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html)
- [Amazon Bedrock abuse detection](https://docs.aws.amazon.com/bedrock/latest/userguide/abuse-detection.html)
- [Model invocation logging in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [AWS Service Terms](https://aws.amazon.com/service-terms/)
