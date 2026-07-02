# Claude Gateway - Sydney Production Deployment

> **⚠️ IMPORTANT: Private Network Requirement**  
> This gateway is deployed on a **private network** using RFC 1918 addresses (`172.16.0.0/16`). Claude Code validates that gateway hostnames resolve to private IP addresses to ensure you're on your organization's internal network. To access this gateway, you must be connected via VPN, have appropriate `/etc/hosts` entries, or use the Cloudflare tunnel as configured below.
>
> **Reference:** [Claude Apps Gateway Documentation](https://code.claude.com/docs/en/claude-apps-gateway)

This document describes the actual production deployment of Claude Gateway in AWS ap-southeast-2 (Sydney).

## Infrastructure Overview

**Region:** `ap-southeast-2` (Sydney)  
**VPC:** `vpc-0c10eba6e8a1e4d3a` (`172.16.0.0/16`)  
**Deployment Date:** 2026-07-02

### Architecture

```
Internet → Cloudflare Tunnel (EC2) → Internal ALB (HTTPS) → EKS Auto Mode → Gateway Pod → Bedrock (ap-southeast-2)
```

## EKS Cluster

**Cluster Name:** `claude-gateway-sydney`  
**Version:** 1.32  
**Type:** EKS Auto Mode (fully managed nodes)  
**Namespace:** `claude-gateway`

### Compute Configuration
- **Node Pools:** `general-purpose` (auto-provisioned on demand)
- **Cluster Role:** `claude-gateway-eks-auto-cluster-role`
- **Node Role:** `claude-gateway-eks-auto-node-role`
- **Current Node:** `i-0b8032a412f986a41` (auto-scaled)

### Networking
**Subnets:**
- Private: `subnet-0a02b6dacab911809` (ap-southeast-2a), `subnet-04429b0499cd83e50` (ap-southeast-2b)
- Public: `subnet-0beb6d0c8f104f389` (ap-southeast-2a), `subnet-0a326e6c4f8dcb84c` (ap-southeast-2b)

**Kubernetes Network:**
- Service CIDR: `10.100.0.0/16`
- Elastic Load Balancing: Enabled
- Block Storage: Enabled

### Access Configuration
- **Authentication Mode:** API (not legacy ConfigMap)
- **API Endpoint:** Public and Private access enabled

## Application Configuration

### Gateway Deployment

**Image:** `331102492406.dkr.ecr.ap-southeast-2.amazonaws.com/claude-gateway:2.1.196`  
**Service Account:** `gateway` (with Pod Identity)  
**Replicas:** 1  

**Resources:**
- Requests: 500m CPU, 512Mi memory
- Limits: 1000m CPU, 1Gi memory

**Health Checks:**
- Readiness: `/readyz` (10s initial delay)
- Liveness: `/healthz` (15s initial delay)

### Authentication & Authorization

**IAM Role:** `claude-gateway-pod-identity-role`  
**Type:** EKS Pod Identity (not IRSA)  
**Association ID:** `a-9losgymtylw7qop7j`

**Permissions:**
- `bedrock:InvokeModel`
- `bedrock:InvokeModelWithResponseStream` 
- `secretsmanager:GetSecretValue`
- `secretsmanager:DescribeSecret`

**OIDC Provider:**
- Trust: Both Pod Identity (`pods.eks.amazonaws.com`) and IRSA (for CSI driver)
- OIDC URL: `https://oidc.eks.ap-southeast-2.amazonaws.com/id/7E71701BFB78EC6EB1B7F111F9478A58`

### Application Load Balancer

**DNS:** `internal-k8s-claudega-claudega-661417b9ce-775087875.ap-southeast-2.elb.amazonaws.com`  
**Private IPs:** `172.16.3.141`, `172.16.18.206`  
**Scheme:** Internal  
**Listener:** HTTPS:443

**Certificate:**
- ARN: `arn:aws:acm:ap-southeast-2:331102492406:certificate/44d60f23-245b-428a-b07b-3bc81a8ff02c`
- Domains: `alickwong.com`, `*.alickwong.com`
- SSL Policy: `ELBSecurityPolicy-TLS13-1-2-2021-06`

**Load Balancer Attributes:**
- Idle timeout: 3600s (for long-running streaming responses)
- Deregistration delay: 30s

**Target Group:**
- Type: IP
- Backend: Pod IP:8080
- Health check: `/readyz`

**Security Groups:**
- `sg-0ac7420d97e819835`: Port 80 from 0.0.0.0/0
- `sg-03445a90fd5439171`: All egress

### Secrets Management

**Provider:** AWS Secrets Manager (ap-southeast-2)  
**CSI Driver:** secrets-store-csi-driver (with AWS provider)  
**Token Audience:** `sts.amazonaws.com`

**Secrets:**
- `claude-gateway/jwt-secret` - JWT signing key
- `claude-gateway/oidc-client-secret` - Google OAuth client secret
- `claude-gateway/postgres-url` - PostgreSQL connection string
- `claude-gateway/admin-read-key` - Admin read API key
- `claude-gateway/admin-write-key` - Admin write API key
- `claude-gateway/config` - Full gateway.yaml configuration

**Mount Points:**
- `/secrets` - Individual secrets (jwt-secret, oidc-client-secret, postgres-url, admin keys)
- `/etc/claude` - gateway.yaml config file

## Database

**Type:** Amazon RDS PostgreSQL 16  
**Instance:** `claude-gateway-db.clunuwhmmnoz.ap-southeast-2.rds.amazonaws.com`  
**Instance Class:** `db.t4g.small`  
**Database Name:** `claude_gateway`  
**Username:** `gateway`  
**Storage:** 20GB encrypted  
**Backup Retention:** 7 days

**Security Group:** `sg-00aa74cf854afa1b9`  
**Allowed CIDR:** `172.16.0.0/16` (entire VPC) on port 5432

**Subnets:** Private subnets only (no public access)

## Gateway Configuration

**Public URL:** `https://claude-gateway.alickwong.com`  
**Trusted Proxies:** `172.16.0.0/16` (VPC CIDR)

### OIDC Authentication
- **Issuer:** `https://accounts.google.com`
- **Client ID:** `366923819918-div2fc4b19g4db6h9fu49ksekp8ljfes.apps.googleusercontent.com`
- **Allowed Domains:** `alickwong.com`
- **Scopes:** `openid`, `profile`, `email`
- **Extra Params:** `access_type: offline`, `prompt: consent`

### Upstream Configuration
- **Provider:** Amazon Bedrock
- **Region:** `ap-southeast-2`
- **Auth:** Pod Identity (no explicit credentials)

### Admin Configuration
- **Read Keys:** `reporting`
- **Write Keys:** `admin`
- **Admin Groups:** `me@alickwong.com`

## External Access

### Cloudflare Tunnel

**EC2 Instance:** `i-00dda1ca375bf9001` (`cloudflare-tunnel-sydney`)  
**Instance Type:** `t2.small`  
**Subnet:** `subnet-0beb6d0c8f104f389` (public subnet in ap-southeast-2a)  
**Private IP:** `172.16.6.117`  
**IAM Profile:** `ec2-jump-host-role`

**Service:** `cloudflared` (systemd service)  
**Origin:** Routes to ALB on HTTPS:443

**Local /etc/hosts:** Maps `claude-gateway.alickwong.com` → `172.16.3.141` (ALB IP)

### Client Configuration

**macOS Managed Settings:**  
File: `/Library/Application Support/ClaudeCode/managed-settings.json`
```json
{
  "forceLoginMethod": "gateway",
  "forceLoginGatewayUrl": "https://claude-gateway.alickwong.com"
}
```

**Local /etc/hosts:**
```
172.16.3.141 claude-gateway.alickwong.com
```

**DNS Resolution:**
- Public DNS (`dig`): Returns public IP (bypasses /etc/hosts)
- System resolver (`dscacheutil`): Returns `172.16.3.141` from /etc/hosts
- Applications use system resolver → connect to ALB

## Management Commands

### Switch to Sydney Cluster
```bash
kubectl config use-context arn:aws:eks:ap-southeast-2:331102492406:cluster/claude-gateway-sydney
```

### View Gateway Logs
```bash
kubectl logs -n claude-gateway -l app=claude-gateway --tail=100 -f
```

### Check Pod Status
```bash
kubectl get pods -n claude-gateway -o wide
```

### Check Ingress/ALB Status
```bash
kubectl get ingress -n claude-gateway
kubectl describe ingress claude-gateway -n claude-gateway
```

### View Pod Identity Association
```bash
aws eks list-pod-identity-associations \
  --cluster-name claude-gateway-sydney \
  --region ap-southeast-2
```

### Access Gateway via Tunnel EC2
```bash
aws ssm start-session \
  --target i-00dda1ca375bf9001 \
  --region ap-southeast-2
```

### Test ALB Health from Tunnel EC2
```bash
aws ssm send-command \
  --instance-ids i-00dda1ca375bf9001 \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["curl -sk https://claude-gateway.alickwong.com/readyz"]' \
  --region ap-southeast-2
```

### Restart Gateway Pod
```bash
kubectl delete pod -n claude-gateway -l app=claude-gateway
```

### Update Gateway Config
1. Update secret in Secrets Manager:
```bash
aws secretsmanager update-secret \
  --secret-id claude-gateway/config \
  --secret-string file://gateway.yaml \
  --region ap-southeast-2
```

2. Restart pod to pick up changes:
```bash
kubectl delete pod -n claude-gateway -l app=claude-gateway
```

### Scale Gateway
```bash
kubectl scale deployment claude-gateway -n claude-gateway --replicas=2
```

## Troubleshooting

### Gateway Not Accessible
1. Check pod is running: `kubectl get pods -n claude-gateway`
2. Check pod logs: `kubectl logs -n claude-gateway -l app=claude-gateway --tail=50`
3. Check ALB target health:
```bash
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn> \
  --region ap-southeast-2
```
4. Verify DNS cache: `dscacheutil -q host -a name claude-gateway.alickwong.com`
5. Flush DNS: `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder`

### "Code Expired" During Login
- Ensure `/etc/hosts` points to `172.16.3.141`
- Flush DNS cache
- Restart Chrome to clear DNS cache
- Check gateway logs for auth errors

### Database Connection Issues
1. Check pod logs for PostgreSQL errors
2. Verify security group allows VPC CIDR on port 5432
3. Test connection from pod:
```bash
kubectl exec -n claude-gateway -it <pod-name> -- sh
# Then inside pod:
curl -v telnet://claude-gateway-db.clunuwhmmnoz.ap-southeast-2.rds.amazonaws.com:5432
```

### Secrets Not Mounting
1. Check CSI driver pods: `kubectl get pods -n kube-system -l app=secrets-store-csi-driver`
2. Verify token requests: `kubectl get csidriver secrets-store.csi.k8s.io -o yaml | grep tokenRequests`
3. Check service account annotation: `kubectl get sa gateway -n claude-gateway -o yaml`
4. Verify IAM role has Secrets Manager permissions

### Bedrock Access Issues
1. Check Pod Identity association exists:
```bash
aws eks describe-pod-identity-association \
  --cluster-name claude-gateway-sydney \
  --association-id a-9losgymtylw7qop7j \
  --region ap-southeast-2
```
2. Verify IAM role has Bedrock permissions
3. Check gateway logs for Bedrock API errors

## Cost Optimization

**Current Monthly Estimate (Sydney):**
- EKS Control Plane: $72/month ($0.10/hour)
- EKS Auto Mode Compute: $0.05/vCPU-hour + $0.007/GB-hour (pay-per-pod)
- RDS db.t4g.small: ~$30/month (20GB storage + 7-day backups)
- ALB: ~$22/month ($0.0225/hour + data processing)
- NAT Gateway: ~$32/month ($0.045/hour + $0.045/GB data)
- Secrets Manager: ~$2.40/month ($0.40/secret × 6)
- EC2 t2.small (tunnel): ~$16/month ($0.023/hour)
- Data Transfer: Variable (S3, Bedrock API calls)

**Total Base Cost:** ~$175-200/month (excluding Bedrock API usage)

## Security Considerations

1. **Network Isolation:** Gateway is internal-only, accessible via Cloudflare tunnel
2. **Encryption:** 
   - TLS 1.3 at ALB (ACM managed certificate)
   - RDS storage encrypted
   - Secrets Manager encrypted at rest
3. **IAM:** Pod Identity with least-privilege Bedrock permissions
4. **Authentication:** Google OIDC with email domain restriction (`alickwong.com`)
5. **Database:** Private subnets only, security group restricted to VPC
6. **Secrets:** Mounted via CSI driver (not environment variables)

## Backup & Disaster Recovery

**RDS Backups:**
- Automated daily backups (7-day retention)
- Manual snapshots available

**Configuration Backup:**
- Secrets stored in Secrets Manager (versioned)
- Gateway config in git repository
- Kubernetes manifests in `/Users/alickw/Documents/Codings/temp/claude-gateway-on-aws/k8s/`

**Recovery Procedure:**
1. Recreate EKS cluster (or use existing)
2. Restore secrets from Secrets Manager
3. Restore RDS from snapshot (or create new with same credentials)
4. Apply Kubernetes manifests
5. Update ALB DNS in `/etc/hosts` and Cloudflare tunnel

## Monitoring

**Gateway Logs:** `kubectl logs -n claude-gateway -l app=claude-gateway -f`

**Key Log Events:**
- `evt:device.authorize` - User authentication attempts
- `evt:auth.denied` - Failed authentication
- `evt:config.load` - Gateway startup/config reload
- `postgres connection closed` - Database connectivity issues

**Metrics:**
- Pod CPU/Memory: `kubectl top pod -n claude-gateway`
- Node resources: `kubectl top nodes`
- ALB metrics: CloudWatch > EC2 > Load Balancers

**Health Checks:**
- Gateway: `https://claude-gateway.alickwong.com/readyz`
- Database: Check pod logs for connection errors

## Related Resources

- Original deployment guide: `/Users/alickw/Documents/Codings/temp/claude-gateway-on-aws/README.md`
- Gateway config: `gateway.yaml` (stored in Secrets Manager)
- Kubernetes manifests: `/Users/alickw/Documents/Codings/temp/claude-gateway-on-aws/k8s/`
- US-East-1 cluster (deprecated): `arn:aws:eks:us-east-1:331102492406:cluster/alick-app-auto`
