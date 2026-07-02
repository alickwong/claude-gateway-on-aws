# Claude Gateway Model Configuration Guide

## Summary

✅ **Issue Resolved**: Claude CLI and Gateway now correctly configured to use AWS Bedrock inference profiles for Claude 4+ models.

## Key Learnings

### AWS Bedrock Model Access Methods

AWS Bedrock provides three ways to access Claude models:

1. **ON_DEMAND** - Direct model access (Claude 3 only)
2. **INFERENCE_PROFILE** - Cross-region routing for high availability (Claude 4+, required)
3. **PROVISIONED** - Reserved capacity (enterprise use)

**Important**: All Claude 4.x models (Opus 4.x, Sonnet 4.x, Haiku 4.5) require **inference profiles**, not direct model IDs.

### Inference Profile Types

| Profile Prefix | Description | Example |
|----------------|-------------|---------|
| `us.anthropic.*` | US cross-region (us-east-1, us-west-2) | `us.anthropic.claude-opus-4-7` |
| `global.anthropic.*` | Global cross-region (worldwide) | `global.anthropic.claude-haiku-4-5-20251001-v1:0` |
| `anthropic.*` | Direct model ID (Claude 3 only) | `anthropic.claude-3-sonnet-20240229-v1:0` |

## Configuration Files

### 1. Claude CLI Settings (`~/.claude/settings.json`)

```json
{
  "fallbackModel": [
    "us.anthropic.claude-sonnet-4-5-20250929-v1:0"
  ],
  "availableModels": [
    "claude-opus-4-7",
    "claude-opus-4-6",
    "claude-opus-4-5",
    "claude-sonnet-4-6",
    "claude-sonnet-4-5",
    "claude-haiku-4-5",
    "claude-3-sonnet",
    "claude-3-haiku"
  ],
  "enforceAvailableModels": true,
  "modelOverrides": {
    "claude-opus-4-7": "us.anthropic.claude-opus-4-7",
    "claude-opus-4-6": "us.anthropic.claude-opus-4-6-v1",
    "claude-opus-4-5": "us.anthropic.claude-opus-4-5-20251101-v1:0",
    "claude-sonnet-4-6": "us.anthropic.claude-sonnet-4-6",
    "claude-sonnet-4-5": "us.anthropic.claude-sonnet-4-5-20250929-v1:0",
    "claude-haiku-4-5": "global.anthropic.claude-haiku-4-5-20251001-v1:0",
    "claude-3-sonnet": "us.anthropic.claude-3-sonnet-20240229-v1:0",
    "claude-3-haiku": "us.anthropic.claude-3-haiku-20240307-v1:0"
  }
}
```

### 2. Gateway Configuration (`gateway.yaml`)

```yaml
listen:
  host: 0.0.0.0
  port: 8080
  public_url: http://localhost:8080
  trusted_proxies: [40.0.0.0/16]

oidc:
  issuer: https://accounts.google.com
  client_id: <your-client-id>
  client_secret: ${file:/secrets/oidc-client-secret}
  allowed_email_domains: [alickwong.com]
  scopes: [openid, profile, email]
  extra_auth_params: { access_type: offline, prompt: consent }

session:
  jwt_secret: ${file:/secrets/jwt-secret}

store:
  postgres_url: ${file:/secrets/postgres-url}

admin:
  read_keys:
    - { id: reporting, key: "${file:/secrets/admin-read-key}" }
  write_keys:
    - { id: admin, key: "${file:/secrets/admin-write-key}" }
  admin_groups: [me@alickwong.com]

upstreams:
  - provider: bedrock
    region: us-west-2
    auth: {}

managed:
  policies:
    operator:
      allowed_models:
        # US cross-region inference profiles (Claude 4+)
        - us.anthropic.claude-opus-4-7
        - us.anthropic.claude-opus-4-6-v1
        - us.anthropic.claude-opus-4-5-20251101-v1:0
        - us.anthropic.claude-sonnet-4-6
        - us.anthropic.claude-sonnet-4-5-20250929-v1:0
        - us.anthropic.claude-3-sonnet-20240229-v1:0
        - us.anthropic.claude-3-haiku-20240307-v1:0
        # Global cross-region inference profiles
        - global.anthropic.claude-haiku-4-5-20251001-v1:0
        - global.anthropic.claude-opus-4-6-v1
        # Legacy on-demand models (Claude 3 only)
        - anthropic.claude-3-sonnet-20240229-v1:0
        - anthropic.claude-3-haiku-20240307-v1:0
```

## Available Models Reference

### Claude 4 Models (via Inference Profiles)

| Model Name | Inference Profile ID | Context Window |
|------------|---------------------|----------------|
| Claude Opus 4.7 | `us.anthropic.claude-opus-4-7` | 1M tokens |
| Claude Opus 4.6 | `us.anthropic.claude-opus-4-6-v1` | 1M tokens |
| Claude Opus 4.5 | `us.anthropic.claude-opus-4-5-20251101-v1:0` | 1M tokens |
| Claude Opus 4.1 | `us.anthropic.claude-opus-4-1-20250805-v1:0` | 1M tokens |
| Claude Sonnet 4.6 | `us.anthropic.claude-sonnet-4-6` | 1M tokens |
| Claude Sonnet 4.5 | `us.anthropic.claude-sonnet-4-5-20250929-v1:0` | 1M tokens |
| Claude Haiku 4.5 | `global.anthropic.claude-haiku-4-5-20251001-v1:0` | 1M tokens |

### Claude 3 Models (Direct Access)

| Model Name | Model ID | Context Window |
|------------|----------|----------------|
| Claude 3 Sonnet | `anthropic.claude-3-sonnet-20240229-v1:0` | 200k tokens |
| Claude 3 Haiku | `anthropic.claude-3-haiku-20240307-v1:0` | 200k tokens |

## Testing Model Access

### Test with AWS CLI

```bash
# Test Sonnet 4.5
aws bedrock-runtime invoke-model \
  --model-id us.anthropic.claude-sonnet-4-5-20250929-v1:0 \
  --region us-west-2 \
  --body '{"anthropic_version":"bedrock-2023-05-31","messages":[{"role":"user","content":"Hi"}],"max_tokens":100}' \
  --cli-binary-format raw-in-base64-out \
  output.json && cat output.json | jq -r '.content[0].text'

# Test Opus 4.7
aws bedrock-runtime invoke-model \
  --model-id us.anthropic.claude-opus-4-7 \
  --region us-west-2 \
  --body '{"anthropic_version":"bedrock-2023-05-31","messages":[{"role":"user","content":"Hi"}],"max_tokens":100}' \
  --cli-binary-format raw-in-base64-out \
  output.json && cat output.json | jq -r '.content[0].text'
```

### List Available Inference Profiles

```bash
# List all Claude inference profiles
aws bedrock list-inference-profiles --region us-west-2 \
  | jq -r '.inferenceProfileSummaries[] | select(.models[0].modelArn | contains("anthropic")) | {id: .inferenceProfileId, name: .inferenceProfileName, type: .type}'

# Check model inference types
aws bedrock list-foundation-models --region us-west-2 \
  | jq -r '.modelSummaries[] | select(.providerName=="Anthropic") | "\(.modelId) - \(.inferenceTypesSupported | join(","))"'
```

## Deploying Gateway Updates

If you have the gateway deployed on Kubernetes:

```bash
# Update the configuration secret
aws secretsmanager update-secret \
  --secret-id claude-gateway/config \
  --secret-string file://gateway.yaml \
  --region us-east-1

# Restart the gateway pods
kubectl rollout restart deployment/claude-gateway -n claude-gateway

# Verify the rollout
kubectl rollout status deployment/claude-gateway -n claude-gateway

# Check logs
kubectl logs -f deployment/claude-gateway -n claude-gateway
```

## Troubleshooting

### Error: "model X is not in the operator's model allowlist"

**Cause**: The model ID in your request doesn't match the `allowed_models` list in `gateway.yaml`.

**Fix**: 
1. Check `modelOverrides` in `~/.claude/settings.json`
2. Verify those model IDs are in `managed.policies.operator.allowed_models` in `gateway.yaml`
3. Restart the gateway after updating

### Error: "Invocation of model ID X with on-demand throughput isn't supported"

**Cause**: Trying to use a direct model ID for a Claude 4 model instead of an inference profile.

**Fix**: Use the inference profile ID (e.g., `us.anthropic.claude-opus-4-7` instead of `anthropic.claude-opus-4-7`)

### Error: "AccessDeniedException: Model access not granted"

**Cause**: You haven't enabled access to the model in the Bedrock console.

**Fix**:
1. Go to AWS Console → Amazon Bedrock → Model access
2. Click "Enable specific models" or "Modify model access"
3. Select the Claude models you need
4. Submit the request (usually instant approval)

### Checking Your Current Model Access

```bash
# List models you have access to
aws bedrock list-foundation-models --region us-west-2 \
  --query 'modelSummaries[?providerName==`Anthropic`].[modelId,modelName,inferenceTypesSupported]' \
  --output table
```

## Best Practices

1. **Use US inference profiles** (`us.anthropic.*`) for best latency if your users are in North America
2. **Use Global inference profiles** (`global.anthropic.*`) for worldwide distribution
3. **Always specify the full version ID** (e.g., `claude-sonnet-4-5-20250929-v1:0`) for reproducibility
4. **Test model access** before deploying to production
5. **Monitor Bedrock quotas** in CloudWatch if you have high traffic
6. **Set ALB idle timeout to 3600s** to prevent streaming response interruptions

## Additional Resources

- [Claude Apps Gateway Documentation](https://code.claude.com/docs/en/claude-apps-gateway)
- [AWS Bedrock Inference Profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html)
- [Claude API Model IDs](https://docs.anthropic.com/en/docs/about-claude/models)
- [Bedrock Model Access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)

## Verification ✅

Both configurations have been tested and confirmed working:

```bash
✓ Claude Sonnet 4.5 works!
✓ Claude Opus 4.7 works!
```

Your Claude CLI is now correctly configured to use AWS Bedrock with inference profiles for all Claude 4+ models.
