# Deploy OpenTelemetry for Claude Code on the Gateway

This guide adds **usage telemetry** to a working [Claude apps gateway](./README.md)
deployment. When it's done, every Claude Code developer's token usage lands in
Amazon CloudWatch, attributed per user, model, and token type — with no
developer-side configuration.

> **Prerequisite:** a running gateway from the [main README](./README.md). This
> guide reuses its cluster, namespace (`claude-gateway`), Secrets Store CSI
> driver, and Pod Identity setup.

## What you'll build

```
                    managed-settings.json (pushed by the gateway per user)
                    sets OTEL_* env vars on each client
                              │
   Claude Code CLI  ──────────┼──────── OTLP/HTTPS ─────────▶  Internal ALB
   stamps user.id /           │         (TLS + Bearer)          otel.<your-domain>
   user.email from the        │                                 (ACM cert)
   gateway JWT                │                                      │
                              │                                      ▼
                                                          OTEL Collector (Deployment)
                                                          bearertokenauth validates
                                                                    │
                                                        ┌───────────┴───────────┐
                                                        ▼                       ▼
                                                  awsemf (EMF)          otlphttp + sigv4
                                                  CloudWatch metrics    CloudWatch
                                                  namespace: ClaudeCode
```

**Design choice — direct-send, not relay.** The gateway *can* relay telemetry
itself (`telemetry.forward_to`), but that couples every client's metrics to the
gateway process. Instead, this guide has the gateway **push `OTEL_*` env vars to
clients via a managed policy**, so each CLI sends OTLP **directly** to a
standalone collector. The CLI still stamps `user.id` / `user.email` /
`user.groups` from its gateway-issued JWT, so per-user attribution is preserved
either way. See [`docs/superpowers/specs/2026-07-03-otel-collector-direct-send-design.md`](./docs/superpowers/specs/2026-07-03-otel-collector-direct-send-design.md)
for the full rationale.

## Prerequisites

- A working gateway deployment (main README, Steps 1–7).
- An **ACM certificate** covering the collector's hostname (e.g. a wildcard
  `*.example.com` that covers `otel.example.com`), in the **same region** as the
  cluster.
- **Terraform** ≥ 1.5 and the AWS CLI, both authenticated to your account.
- Private subnets tagged/known for an **internal** ALB (reuse the same subnets
  as the gateway ingress).

Set these shell variables — every step below uses them:

```bash
export AWS_REGION=ap-southeast-2
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME=claude-gateway-sydney
export OTEL_HOST=otel.example.com                     # collector hostname (must match the ACM cert)
export ACM_CERT_ARN=arn:aws:acm:${AWS_REGION}:${AWS_ACCOUNT_ID}:certificate/<your-cert-id>
export ALB_SUBNETS=subnet-aaaa,subnet-bbbb            # same private subnets as the gateway ingress
```

## Step 1: Create the bearer token secret

Clients authenticate to the collector with a bearer token. Generate one and
store it in Secrets Manager (the same store the gateway already uses):

```bash
aws secretsmanager create-secret \
  --region "$AWS_REGION" \
  --name claude-gateway/otel-bearer-token \
  --description "Bearer token clients present to the OTEL collector ALB" \
  --secret-string "$(openssl rand -hex 32)"
```

> **Rotation:** the collector reads this token from a mounted file, and the
> gateway injects the same token into clients. To rotate, update the secret
> value, then restart both the collector and gateway pods so they re-read it.

## Step 2: Provision IAM with Terraform

The collector's pods need an IAM role that can write to CloudWatch and read the
bearer token. The Terraform in [`terraform/otel-collector/`](./terraform/otel-collector/)
creates the role, its policy, and the EKS Pod Identity association.

```bash
cd terraform/otel-collector

terraform init

terraform apply \
  -var "region=${AWS_REGION}" \
  -var "cluster_name=${CLUSTER_NAME}"

# Note the output — you'll use it in Step 3.
terraform output collector_role_arn
```

This creates the role `claude-otel-collector-pod-identity-role`, trusting **both**
EKS Pod Identity and IRSA (the Secrets Store CSI provider resolves the role via
the IRSA annotation, while the collector runtime uses Pod Identity for
CloudWatch — the role must trust both). Its policy grants:

- `cloudwatch:PutMetricData` (for the `otlphttp` sigv4 exporter)
- `logs:*` scoped to `/aws/claude-code/*` (for the `awsemf` EMF exporter)
- `secretsmanager:GetSecretValue` scoped to `claude-gateway/otel-bearer-token-*`

## Step 3: Deploy the collector

The manifest [`k8s/otel-collector.yaml`](./k8s/otel-collector.yaml) contains the
ServiceAccount, SecretProviderClass, ConfigMap (collector pipeline), Deployment,
Service, and the internal ALB Ingress.

**Before applying, replace the reference values** with your own. The committed
file is pinned to the reference deployment (account `331102492406`, region
`ap-southeast-2`, a specific cert ARN, subnets, and `otel.alickwong.com`):

| Location in `k8s/otel-collector.yaml` | Replace with |
| --- | --- |
| SA annotation `eks.amazonaws.com/role-arn` | your `collector_role_arn` from Step 2 |
| ConfigMap `sigv4auth.region`, `awsemf.region`, `otlphttp.endpoint` | your `$AWS_REGION` |
| ConfigMap `resource.aws.account_id` value | your `$AWS_ACCOUNT_ID` |
| Ingress `certificate-arn` | your `$ACM_CERT_ARN` |
| Ingress `subnets` | your `$ALB_SUBNETS` |
| Ingress `rules[].host` | your `$OTEL_HOST` |

A quick `sed` for the common ones (review the diff before applying):

```bash
sed -e "s/331102492406/${AWS_ACCOUNT_ID}/g" \
    -e "s/ap-southeast-2/${AWS_REGION}/g" \
    -e "s#certificate/44d60f23-245b-428a-b07b-3bc81a8ff02c#certificate/<your-cert-id>#g" \
    -e "s/subnet-0beb6d0c8f104f389,subnet-0a326e6c4f8dcb84c/${ALB_SUBNETS}/g" \
    -e "s/otel.alickwong.com/${OTEL_HOST}/g" \
    k8s/otel-collector.yaml | kubectl apply -f -
```

Wait for the collector to be ready:

```bash
kubectl rollout status deploy/otel-collector -n claude-gateway --timeout=120s
kubectl logs -n claude-gateway -l app=otel-collector --tail=20
# Expect: "Everything is ready. Begin running and processing data."
```

> **Collector image:** this uses `otel/opentelemetry-collector-contrib`, **not**
> the AWS-curated ADOT image. ADOT does not bundle the `bearertokenauth`
> extension; the contrib distro includes `bearertokenauth` + `sigv4auth` +
> `awsemf`, all three of which this pipeline needs.

## Step 4: Verify the ALB and TLS

```bash
# The ALB hostname:
ALB=$(kubectl get ingress otel-collector -n claude-gateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "$ALB"

# Confirm the HTTPS:443 listener and a healthy target:
LBARN=$(aws elbv2 describe-load-balancers --region "$AWS_REGION" \
  --query "LoadBalancers[?DNSName=='$ALB'].LoadBalancerArn" --output text)
aws elbv2 describe-listeners --region "$AWS_REGION" --load-balancer-arn "$LBARN" \
  --query "Listeners[].{port:Port,proto:Protocol}" --output table
TG=$(aws elbv2 describe-target-groups --region "$AWS_REGION" --load-balancer-arn "$LBARN" \
  --query "TargetGroups[0].TargetGroupArn" --output text)
aws elbv2 describe-target-health --region "$AWS_REGION" --target-group-arn "$TG" \
  --query "TargetHealthDescriptions[].TargetHealth.State" --output text
# Expect: healthy
```

## Step 5: Point the gateway at the collector

Update the gateway's `gateway.yaml` (stored in Secrets Manager as
`claude-gateway/config`) to **remove any `telemetry:` relay block** and add a
`managed` policy that pushes the direct-send `OTEL_*` env vars to clients. See
[`gateway.template.yaml`](./gateway.template.yaml) for the exact block:

```yaml
managed:
  policies:
    - match: {}                       # catch-all: applies to every authenticated user
      cli:
        env:
          CLAUDE_CODE_ENABLE_TELEMETRY: "1"
          OTEL_METRICS_EXPORTER: otlp
          OTEL_EXPORTER_OTLP_PROTOCOL: http/protobuf
          OTEL_EXPORTER_OTLP_ENDPOINT: https://otel.example.com      # your $OTEL_HOST
          OTEL_EXPORTER_OTLP_HEADERS: "Authorization=Bearer ${file:/secrets/otel-bearer-token}"
```

The gateway needs the bearer token mounted for the `${file:...}` expansion. Add
it to the gateway's SecretProviderClass (see
[`k8s/secret-provider-class.yaml`](./k8s/secret-provider-class.yaml), which
already includes the `otel-bearer-token` object), then push the config and roll
the gateway:

```bash
# Push the updated config (assumes you have your gateway.yaml locally):
aws secretsmanager put-secret-value --region "$AWS_REGION" \
  --secret-id claude-gateway/config \
  --secret-string "$(cat gateway.yaml)"

# Ensure the bearer token is mounted into the gateway pod:
kubectl apply -f k8s/secret-provider-class.yaml

# Roll the gateway to pick up the new config + mount:
kubectl rollout restart deploy/claude-gateway -n claude-gateway
kubectl rollout status  deploy/claude-gateway -n claude-gateway --timeout=150s

# Confirm a clean boot:
kubectl logs -n claude-gateway -l app=claude-gateway -c gateway --tail=20 \
  | grep -E "config.load|managed settings|telemetry relay"
# Expect: "telemetry relay: not configured" and "managed settings: configured"
```

> **The gateway's IAM role must be able to read the new secret.** The main-README
> `secrets-manager-read` policy is scoped to `claude-gateway/*`, which already
> covers `claude-gateway/otel-bearer-token`. If you scoped it more tightly, widen
> it.

## Step 6: Make the collector hostname resolve (client side)

Claude Code resolves `$OTEL_HOST` through the OS resolver and must reach the
**internal** ALB over your private network (VPN, split tunnel, Direct Connect,
etc.). Because the ALB is internal, the hostname must resolve to its **private**
IPs. Pick one:

- **Private Route 53 zone** (recommended): create an ALIAS record for
  `$OTEL_HOST` → the ALB, in a private hosted zone associated with the VPC your
  clients route into. Durable and handles ALB IP changes automatically.
- **`/etc/hosts`** (quick, per-machine): map the hostname to one of the ALB's
  private IPs. Static — if the ALB is recreated its IPs change, and you must
  update the entry.

```bash
# Resolve the ALB's current private IPs:
dig +short "$ALB"

# /etc/hosts option (macOS/Linux):
echo "<one-of-those-private-ips> ${OTEL_HOST}" | sudo tee -a /etc/hosts
```

> **Why not public DNS?** Claude Code's `/login` and gateway model require the
> gateway to resolve to private addresses; the same private-network posture
> applies here. Keeping the collector internal means the bearer token is
> defense-in-depth, not the only control.

## Step 7: Roll it out to developers

The gateway delivers the `OTEL_*` env vars through **managed settings**, which
each client fetches on its next poll (within ~1 hour) or on the next
`claude` restart / `/login`. Because these settings can influence the client,
Claude Code shows a **one-time approval dialog** the first time — developers must
accept it.

To verify on your own machine, restart Claude Code, use it briefly, then check
CloudWatch (allow ~1–2 minutes for the [metric delay](#metric-delay)):

```bash
aws cloudwatch list-metrics --region "$AWS_REGION" --namespace ClaudeCode \
  --query "Metrics[].{M:MetricName,D:Dimensions}" --output json
```

## Viewing metrics

Console: **CloudWatch → Metrics → All metrics → Custom namespaces →
`ClaudeCode`** → pick a dimension set → select `claude_code.token.usage` → set
the statistic to **Sum**.

| Dimension set | What you get |
| --- | --- |
| `[user.email]` | Total tokens per user |
| `[user.email, model, type]` | Per-user, per-model, input vs output (most granular) |
| `[model]` | Tokens per model across all users |

CLI example — token usage for one user over the last day:

```bash
aws cloudwatch get-metric-statistics --region "$AWS_REGION" \
  --namespace ClaudeCode --metric-name claude_code.token.usage \
  --dimensions Name=user.email,Value=dev@example.com \
  --start-time "$(date -u -v-1d +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 3600 --statistics Sum
```

> **Always use the `Sum` statistic.** `claude_code.token.usage` is a monotonic
> counter; the collector emits deltas. A cumulative counter's **first**
> datapoint establishes a baseline and emits nothing — you need at least two
> points before a value appears. (This is why a single test request may look
> like "no data.")

## Metric delay

End-to-end latency is roughly **1–2.5 minutes**, the sum of:

| Stage | Delay | Why |
| --- | --- | --- |
| Client export interval | 0–60s | Claude Code exports every 60s (`OTEL_METRIC_EXPORT_INTERVAL` default) |
| Collector batch | 0–60s | `batch/metrics.timeout: 60s` in the ConfigMap |
| EMF → CloudWatch ingestion | ~5–15s | EMF log event parsed into a metric |

To tighten to ~20–30s, push `OTEL_METRIC_EXPORT_INTERVAL=10000` in the managed
policy `env` and set `batch/metrics.timeout: 10s` in the ConfigMap (at the cost
of more, smaller CloudWatch writes).

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Collector pod stuck `ContainerCreating`, event `An IAM role must be associated with service account` | The SA lacks the IRSA annotation, or the role doesn't trust IRSA | Ensure the SA has `eks.amazonaws.com/role-arn` and the Terraform role includes the `IRSA` trust statement (it does by default) |
| Collector event `Failed to fetch secret ... Verify secret exists and required permissions` | Role can't read the bearer token | Confirm Step 1 created `claude-gateway/otel-bearer-token` and the Terraform `SecretsManagerRead` statement covers it |
| Collector boots but `bearertokenauth` is an `unknown type` | Using the ADOT image instead of contrib | Use `otel/opentelemetry-collector-contrib` (Step 3 note) |
| Client POST returns `401` | Missing/wrong bearer token | The gateway pushes the token via managed settings; ensure the client accepted the approval dialog and the gateway config expands `${file:/secrets/otel-bearer-token}` |
| TLS error / hostname mismatch on the client | ACM cert doesn't cover `$OTEL_HOST`, or DNS points elsewhere | Cert SAN must cover the hostname; `$OTEL_HOST` must resolve to the ALB's private IPs (Step 6) |
| `list-metrics` empty after a single test request | Cumulative-counter baselining | Send at least two increasing datapoints, or just use the CLI normally for a couple of minutes |
| ALB target `unhealthy` | Health check misconfigured | The Ingress health-checks port `13133` path `/`; confirm the collector's `health_check` extension is listening |
| Gateway boot fails after the config change | `${file:/secrets/otel-bearer-token}` not mounted | Apply `k8s/secret-provider-class.yaml` (includes the token) **before** rolling the gateway |

## Rollback

To revert to gateway-relayed telemetry (or disable it entirely): remove the
`managed.policies` `env` OTEL vars from `gateway.yaml`, optionally restore a
`telemetry.forward_to` block, and roll the gateway. The collector Deployment and
ALB can stay running (harmless if no client points at them) or be removed with
`kubectl delete -f k8s/otel-collector.yaml` and `terraform destroy`.
