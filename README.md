# Deploy Claude Apps Gateway on AWS

> A worked example of running Claude apps gateway on AWS: EKS (or ECS Fargate), Amazon RDS for PostgreSQL, AWS Secrets Manager, and IAM Roles for Service Accounts (IRSA) auth to Amazon Bedrock.

> **Note:** This page walks through one way to run Claude apps gateway on AWS. The configuration is a working example for customer-managed infrastructure rather than a supported production deployment; use it to see how the pieces fit together before adapting it to your own environment. For the platform-agnostic requirements, see the [deployment guide](https://code.claude.com/docs/en/claude-apps-gateway-deploy).

This example provisions Claude apps gateway on AWS with Amazon Bedrock as the model upstream, using EKS for compute. Any OpenID Connect (OIDC) compliant IdP works for authentication; only the `oidc` block changes. See [Identity provider setup](https://code.claude.com/docs/en/claude-apps-gateway-deploy#identity-provider-setup) for per-IdP details.

## What you'll build

The reference configuration provisions:

* **Amazon EKS** cluster running the gateway container (with ECS Fargate as an alternative)
* **Amazon ECR** repository for the gateway image
* **Amazon RDS for PostgreSQL** instance, private subnets only, for the gateway's [store](https://code.claude.com/docs/en/claude-apps-gateway-config#store)
* **AWS Secrets Manager** secrets for `gateway.yaml`, the JWT signing key, the OIDC client secret, and the Postgres URL
* **IAM Role** with `bedrock:InvokeModel*` permissions, bound via IRSA on EKS (or task role on ECS)
* **Internal Application Load Balancer (ALB)** for HTTPS termination

## Prerequisites

* An AWS account with permission to create the resources above
* The `aws` CLI configured and authenticated, `kubectl`, `helm`, and Docker installed locally
* For the EKS track: an EKS cluster (or you'll create one below)
* Access to the Claude models you need in Amazon Bedrock, enabled in the target region
* An OIDC provider (e.g., Okta, Azure AD, Google Workspace) with a web-application client and redirect URI `https://<gateway-host>/oauth/callback`
* A TLS hostname for the gateway (internal DNS name pointing at the ALB) and an ACM certificate
* A VPC with at least 2 private subnets across different AZs

Set the account and region once:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=ap-southeast-2   # a region where the Claude models you need are available in Bedrock
export VPC_ID=<your-vpc-id>
export PRIVATE_SUBNET_IDS=<subnet-1-id>,<subnet-2-id>   # private subnets for RDS and EKS
export CLUSTER_NAME=claude-gateway-cluster
```

## Deploy the gateway

### Step 1: Create the ECR repository and push the image

```bash
aws ecr create-repository \
  --repository-name claude-gateway \
  --region "$AWS_REGION" \
  --image-scanning-configuration scanOnPush=true

aws ecr get-login-password --region "$AWS_REGION" | \
  docker login --username AWS --password-stdin "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

# Build the image per the container image requirements using the linux-x64 glibc binary
docker build --platform=linux/amd64 \
  -t "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/claude-gateway:<version>" .
docker push "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/claude-gateway:<version>"
```

### Step 2: Provision Amazon RDS for PostgreSQL

Create a DB subnet group and a PostgreSQL instance in private subnets only:

```bash
# Security group for RDS — allows inbound from EKS pods
aws ec2 create-security-group \
  --group-name claude-gateway-rds-sg \
  --description "Claude gateway RDS access" \
  --vpc-id "$VPC_ID" \
  --output text --query 'GroupId'
# Save the output as RDS_SG_ID

RDS_SG_ID=<output-from-above>

# Allow PostgreSQL traffic from EKS pod CIDR or the EKS security group
aws ec2 authorize-security-group-ingress \
  --group-id "$RDS_SG_ID" \
  --protocol tcp --port 5432 \
  --source-group <eks-node-or-pod-security-group-id>

# Create subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name claude-gateway-db-subnet \
  --db-subnet-group-description "Private subnets for Claude gateway DB" \
  --subnet-ids <subnet-1-id> <subnet-2-id>

# Generate a password
PGPASS="$(openssl rand -hex 24)"

# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier claude-gateway-db \
  --db-instance-class db.t4g.small \
  --engine postgres \
  --engine-version 16 \
  --master-username gateway \
  --master-user-password "$PGPASS" \
  --allocated-storage 20 \
  --storage-encrypted \
  --no-publicly-accessible \
  --vpc-security-group-ids "$RDS_SG_ID" \
  --db-subnet-group-name claude-gateway-db-subnet \
  --db-name claude_gateway \
  --backup-retention-period 7

# Wait for the instance to become available
aws rds wait db-instance-available --db-instance-identifier claude-gateway-db

# Get the endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier claude-gateway-db \
  --query 'DBInstances[0].Endpoint.Address' --output text)

GATEWAY_POSTGRES_URL="postgres://gateway:${PGPASS}@${RDS_ENDPOINT}:5432/claude_gateway?sslmode=require"
```

### Step 3: Create the IAM role for the gateway

Create an IAM role the gateway pod assumes to (1) invoke Bedrock models and (2) let the Secrets Store CSI driver read the gateway's secrets. This guide uses **EKS Pod Identity** (the current recommended mechanism); see the note at the end of this step for the IRSA alternative.

> **Important — inference profiles:** Claude Code invokes models through Bedrock **inference profiles** (e.g. `au.anthropic.claude-opus-4-6-v1`, `global.anthropic.claude-opus-4-8`), not bare foundation-model IDs. When you invoke via an inference profile, Bedrock authorizes the request against **both** the `inference-profile/*` ARN **and** the underlying `foundation-model/*` ARNs in every region the profile can route to. A policy that only grants `foundation-model/anthropic.*` will fail with `403 ... is not authorized to perform: bedrock:InvokeModelWithResponseStream on resource: arn:aws:bedrock:<region>:<account>:inference-profile/...`. You must grant both resource types.

```bash
# Create the IAM policy for Bedrock access.
# NOTE: covers foundation models AND inference profiles (required for
# cross-region / geo inference profiles like au.* and global.*).
cat > /tmp/bedrock-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvokeFoundationModels",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.*"
    },
    {
      "Sid": "InvokeInferenceProfiles",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:*:${AWS_ACCOUNT_ID}:inference-profile/*",
        "arn:aws:bedrock:*:${AWS_ACCOUNT_ID}:application-inference-profile/*"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name claude-gateway-bedrock-access \
  --policy-document file:///tmp/bedrock-policy.json

BEDROCK_POLICY_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:policy/claude-gateway-bedrock-access"

# Policy allowing the Secrets Store CSI driver to read the gateway's secrets.
# Without this the pod's secret/config volumes fail to mount.
cat > /tmp/secrets-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:claude-gateway/*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name claude-gateway-secrets-access \
  --policy-document file:///tmp/secrets-policy.json

SECRETS_POLICY_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:policy/claude-gateway-secrets-access"

# Trust policy for EKS Pod Identity — the pod identity agent assumes this role.
cat > /tmp/trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
EOF

aws iam create-role \
  --role-name claude-gateway-pod-identity-role \
  --assume-role-policy-document file:///tmp/trust-policy.json

aws iam attach-role-policy \
  --role-name claude-gateway-pod-identity-role \
  --policy-arn "$BEDROCK_POLICY_ARN"

aws iam attach-role-policy \
  --role-name claude-gateway-pod-identity-role \
  --policy-arn "$SECRETS_POLICY_ARN"
```

> **IRSA alternative:** If you prefer IAM Roles for Service Accounts instead of Pod Identity, use this trust policy on the role instead of the one above (requires an IAM OIDC provider on the cluster), and skip the `create-pod-identity-association` in Step 6a:
>
> ```bash
> OIDC_PROVIDER=$(aws eks describe-cluster --name "$CLUSTER_NAME" \
>   --query "cluster.identity.oidc.issuer" --output text | sed 's|https://||')
> # Principal: { "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}" }
> # Action: "sts:AssumeRoleWithWebIdentity"
> # Condition StringEquals:
> #   "${OIDC_PROVIDER}:sub": "system:serviceaccount:claude-gateway:gateway"
> #   "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
> ```

### Step 4: Store secrets in AWS Secrets Manager

```bash
# JWT signing secret
JWT_SECRET=$(openssl rand -base64 32)
aws secretsmanager create-secret \
  --name claude-gateway/jwt-secret \
  --secret-string "$JWT_SECRET"

# OIDC client secret (from your IdP)
aws secretsmanager create-secret \
  --name claude-gateway/oidc-client-secret \
  --secret-string "<your-oidc-client-secret>"

# Postgres URL
aws secretsmanager create-secret \
  --name claude-gateway/postgres-url \
  --secret-string "$GATEWAY_POSTGRES_URL"

# Admin API keys — read-only (reporting) and read-write (admin)
aws secretsmanager create-secret \
  --name claude-gateway/admin-read-key \
  --secret-string "$(openssl rand -base64 32)"
aws secretsmanager create-secret \
  --name claude-gateway/admin-write-key \
  --secret-string "$(openssl rand -base64 32)"

# Full gateway.yaml config (created in next step)
aws secretsmanager create-secret \
  --name claude-gateway/config \
  --secret-string file://gateway.yaml
```

### Step 5: Write gateway.yaml

The `upstreams` block points at Amazon Bedrock with `auth: {}`, so the gateway authenticates via the pod's IAM credentials (Pod Identity or IRSA). See the [configuration reference](https://code.claude.com/docs/en/claude-apps-gateway-config) for every field.

```yaml
# gateway.yaml — Google Workspace example
listen:
  host: 0.0.0.0
  port: 8080
  public_url: https://claude-gateway.internal.example.com
  trusted_proxies: [10.0.0.0/8]   # VPC CIDR / ALB subnet range

oidc:
  issuer: https://accounts.google.com
  client_id: <your-oauth-client-id>
  client_secret: ${file:/secrets/oidc-client-secret}
  allowed_email_domains: [example.com]
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
  admin_groups: [<your-admin-email-or-group>]

upstreams:
  - provider: bedrock
    region: ap-southeast-2         # must match $AWS_REGION
    auth: {}                       # uses the pod's IAM credentials (Pod Identity / IRSA)
```

> **Note:** Adjust the `oidc` block for your identity provider. The example shows Google Workspace. For Azure AD, use `issuer: https://login.microsoftonline.com/<tenant-id>/v2.0` and add `offline_access` to `scopes` instead of `extra_auth_params`. For Okta, use `issuer: https://<your-org>.okta.com`. The `extra_auth_params` field passes additional parameters to the authorization endpoint — Google requires `access_type: offline` to issue refresh tokens.

### Step 6: Deploy on EKS

#### 6a. Set up the namespace and service account

```bash
kubectl create namespace claude-gateway

kubectl create serviceaccount gateway -n claude-gateway

# Associate the service account with the IAM role via EKS Pod Identity.
# Requires the "Amazon EKS Pod Identity Agent" add-on on the cluster:
#   aws eks create-addon --cluster-name "$CLUSTER_NAME" --addon-name eks-pod-identity-agent
aws eks create-pod-identity-association \
  --cluster-name "$CLUSTER_NAME" \
  --namespace claude-gateway \
  --service-account gateway \
  --role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/claude-gateway-pod-identity-role"
```

> **Using IRSA instead?** Skip the association above and annotate the service account with the role ARN:
> ```bash
> kubectl annotate serviceaccount gateway -n claude-gateway \
>   eks.amazonaws.com/role-arn="arn:aws:iam::${AWS_ACCOUNT_ID}:role/claude-gateway-pod-identity-role"
> ```

#### 6b. Install the AWS Secrets Manager CSI driver

```bash
# Install the Secrets Store CSI driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true

# Install the AWS provider
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

#### 6c. Create the SecretProviderClasses

The gateway uses two separate SecretProviderClasses: one for secrets (mounted at `/secrets`) and one for the config file (mounted at `/etc/claude`):

```yaml
# secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: claude-gateway-secrets
  namespace: claude-gateway
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "claude-gateway/jwt-secret"
        objectType: "secretsmanager"
        objectAlias: "jwt-secret"
      - objectName: "claude-gateway/oidc-client-secret"
        objectType: "secretsmanager"
        objectAlias: "oidc-client-secret"
      - objectName: "claude-gateway/postgres-url"
        objectType: "secretsmanager"
        objectAlias: "postgres-url"
      - objectName: "claude-gateway/admin-read-key"
        objectType: "secretsmanager"
        objectAlias: "admin-read-key"
      - objectName: "claude-gateway/admin-write-key"
        objectType: "secretsmanager"
        objectAlias: "admin-write-key"
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: claude-gateway-config
  namespace: claude-gateway
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "claude-gateway/config"
        objectType: "secretsmanager"
        objectAlias: "gateway.yaml"
```

```bash
kubectl apply -f secret-provider-class.yaml
```

#### 6d. Deploy the gateway

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: claude-gateway
  namespace: claude-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: claude-gateway
  template:
    metadata:
      labels:
        app: claude-gateway
    spec:
      serviceAccountName: gateway
      containers:
        - name: gateway
          image: <account-id>.dkr.ecr.<region>.amazonaws.com/claude-gateway:<version>
          ports:
            - containerPort: 8080
              protocol: TCP
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          volumeMounts:
            - name: secrets
              mountPath: /secrets
              readOnly: true
            - name: config
              mountPath: /etc/claude
              readOnly: true
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: claude-gateway-secrets
        - name: config
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: claude-gateway-config
---
apiVersion: v1
kind: Service
metadata:
  name: claude-gateway
  namespace: claude-gateway
spec:
  type: ClusterIP
  selector:
    app: claude-gateway
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
```

```bash
kubectl apply -f deployment.yaml
```

#### 6e. Create the internal ALB Ingress

Install the AWS Load Balancer Controller if not already present:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName="$CLUSTER_NAME" \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Create the Ingress resource:

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: claude-gateway
  namespace: claude-gateway
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: <your-acm-certificate-arn>
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/healthcheck-path: /readyz
    alb.ingress.kubernetes.io/target-group-attributes: deregistration_delay.timeout_seconds=30
    alb.ingress.kubernetes.io/load-balancer-attributes: idle_timeout.timeout_seconds=3600
    alb.ingress.kubernetes.io/scheme: internal   # internal-only ALB (no public IP)
    alb.ingress.kubernetes.io/subnets: <subnet-1-id>,<subnet-2-id>
spec:
  ingressClassName: alb
  rules:
    - host: claude-gateway.internal.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: claude-gateway
                port:
                  number: 8080
```

```bash
kubectl apply -f ingress.yaml
```

> **Important:** Set `idle_timeout.timeout_seconds=3600` on the ALB to prevent long streaming responses from being cut off. The default 60-second timeout will terminate active SSE streams.

### Step 7: Push the gateway URL to developer machines

The gateway is now running. Set `forceLoginMethod` and `forceLoginGatewayUrl` in the [managed settings file](https://code.claude.com/docs/en/claude-apps-gateway#set-the-gateway-url) you deploy to each device via MDM. There is no gateway option in the login picker for a developer to select manually.

## Alternative: ECS Fargate deployment

If you prefer ECS Fargate over EKS:

```bash
# Create ECS cluster
aws ecs create-cluster --cluster-name claude-gateway

# Create task execution role (for pulling images and secrets)
# Create task role (for Bedrock access — attach the bedrock policy)

# Create task definition
cat > /tmp/task-def.json << EOF
{
  "family": "claude-gateway",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/claude-gateway-execution-role",
  "taskRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/claude-gateway-task-role",
  "containerDefinitions": [
    {
      "name": "gateway",
      "image": "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/claude-gateway:<version>",
      "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/readyz || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 15
      },
      "secrets": [
        {
          "name": "GATEWAY_JWT_SECRET",
          "valueFrom": "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:claude-gateway/jwt-secret"
        },
        {
          "name": "OIDC_CLIENT_SECRET",
          "valueFrom": "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:claude-gateway/oidc-client-secret"
        },
        {
          "name": "GATEWAY_POSTGRES_URL",
          "valueFrom": "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:claude-gateway/postgres-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/claude-gateway",
          "awslogs-region": "${AWS_REGION}",
          "awslogs-stream-prefix": "gateway"
        }
      }
    }
  ]
}
EOF

aws ecs register-task-definition --cli-input-json file:///tmp/task-def.json

# Create the service with an internal ALB
aws ecs create-service \
  --cluster claude-gateway \
  --service-name claude-gateway \
  --task-definition claude-gateway \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<subnet-1>,<subnet-2>],securityGroups=[<sg-id>],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=<target-group-arn>,containerName=gateway,containerPort=8080"
```

For ECS Fargate, `gateway.yaml` references environment variables (`${GATEWAY_JWT_SECRET}`, `${OIDC_CLIENT_SECRET}`, `${GATEWAY_POSTGRES_URL}`) since secrets inject as env vars via the task definition.

## GCP-to-AWS mapping reference

| GCP concept | AWS equivalent |
|---|---|
| Cloud Run | ECS Fargate |
| GKE | EKS |
| Artifact Registry | Amazon ECR |
| Cloud SQL for PostgreSQL | Amazon RDS for PostgreSQL |
| Secret Manager | AWS Secrets Manager |
| Service Account + Workload Identity | IAM Role + IRSA |
| Agent Platform (Vertex AI) | Amazon Bedrock |
| Internal Application Load Balancer | Internal ALB (via AWS Load Balancer Controller) |
| Private Services Access (VPC peering) | VPC private subnets + security groups |
| `roles/aiplatform.user` | `bedrock:InvokeModel*` IAM policy |
| `gcloud` CLI | `aws` CLI |
| Domain Restricted Sharing | SCPs (Service Control Policies) |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Pod fails to start with `AccessDenied` calling Bedrock | IRSA not configured correctly; pod not using annotated service account | Verify the SA annotation matches the IAM role ARN, the trust policy references the correct OIDC provider and namespace/SA, and the EKS OIDC provider is registered in IAM |
| Gateway boot exits with a Postgres connection-timeout error | Security group doesn't allow ingress from EKS pods on port 5432, or RDS is not in the same VPC | Ensure the RDS security group allows TCP 5432 from the EKS pod security group or node CIDR |
| ALB returns 502 Bad Gateway | Target group health check failing; gateway not ready | Check the health check path is `/readyz`, and the target group port matches 8080 |
| Streaming responses cut off after 60 seconds | ALB idle timeout is at the default 60s | Set `idle_timeout.timeout_seconds=3600` on the ALB via the Ingress annotation |
| OIDC callback fails with redirect_uri mismatch | The `public_url` in gateway.yaml doesn't match the registered redirect URI in your IdP | Ensure `listen.public_url` matches the ALB hostname and the IdP's redirect URI is `<public_url>/oauth/callback` |
| Pod can't pull image from ECR | Node/pod IAM role missing ECR permissions | Attach `AmazonEC2ContainerRegistryReadOnly` to the node instance role or create an ECR pull-through cache |
| Secrets not mounting in pod | Secrets Store CSI driver or AWS provider not installed; SecretProviderClass references wrong secret names | Verify the CSI driver pods are running, and the gateway service account's IAM role has `secretsmanager:GetSecretValue` permission |

## Next steps

* [Configuration reference](https://code.claude.com/docs/en/claude-apps-gateway-config): every `gateway.yaml` option, including `managed.policies` and `telemetry`
* [Deployment and operations](https://code.claude.com/docs/en/claude-apps-gateway-deploy): IdP setup, health checks, JWT secret rotation, upgrades, and the security model
* [Claude apps gateway overview](https://code.claude.com/docs/en/claude-apps-gateway): quickstart and connecting developers
