# STIGman EKS Lab

A self-contained AWS lab that deploys [STIG Manager](https://github.com/NUWCDIVNPT/stig-manager) on EKS — a DoD/RMF-aligned STIG compliance management platform backed by Keycloak for OIDC authentication.

The lab spins up two isolated EKS clusters inside a shared VPC, accessible only via a Windows jump box. Everything is internal except the RDP entry point.

---

## Architecture

```
AWS VPC 10.100.0.0/16 (us-east-2)
│
├── Public Subnet (10.100.11.0/24, 10.100.12.0/24)
│   ├── Windows jump box (t3.medium) — RDP :3389
│   └── NAT Gateway (single AZ)
│
└── Private Subnets (10.100.1.0/24, 10.100.2.0/24)
    │
    ├── keycloak-eks (EKS 1.32, t3.medium ×1)
    │   ├── Internal ALB (HTTPS :443 / HTTP :80)
    │   └── Keycloak pod (nuwcdivnpt/stig-manager-auth)
    │
    └── stigman-eks (EKS 1.32, t3.medium ×2)
        ├── Internal ALB (HTTPS :443 / HTTP :80) ← OIDC provider: keycloak-eks ALB
        ├── STIGman API pod (nuwcdivnpt/stig-manager :54000)
        ├── MySQL pod (mysql:8, :3306)
        └── EBS CSI driver + 20Gi gp2 volume
```

**Traffic flow:**

1. You RDP from your Linux VM (`xfreerdp`) into the Windows jump box.
2. From Edge inside the RDP session, you hit the STIGman internal ALB over HTTPS.
3. STIGman API redirects your browser to Keycloak (OIDC) for authentication.
4. Keycloak issues a JWT; STIGman validates it, then serves the UI.


---

## Prerequisites

| Tool | Version |
|---|---|
| Terraform | >= 1.5 |
| AWS CLI v2 | latest |
| kubectl | >= 1.28 |
| helm | >= 3.12 |

AWS credentials need `AdministratorAccess` (or scoped EKS + EC2 + IAM + ACM + ELB permissions).



## Deployment — Terraform (fast path)

### 1. Initialize and apply infrastructure

```bash
cd terraform/
terraform init
terraform apply
```

Terraform provisions:
- VPC with public/private subnets, NAT gateway, subnet tags for EKS
- `keycloak-eks` cluster (EKS 1.32, 1× t3.medium node group)
- `stigman-eks` cluster (EKS 1.32, 2× t3.medium node group, EBS CSI add-on)
- IAM roles: `eks-cluster-role`, `eks-node-role`, `eks-ebs-csi-stigman`, `eks-lbc-keycloak`, `eks-lbc-stigman`
- Shared `AWSLoadBalancerControllerIAMPolicy`
- Windows jump box (t3.medium) with RDP security group

Outputs you'll need later:
```
keycloak_cluster_name     = keycloak-eks
stigman_cluster_name      = stigman-eks
windows_public_ip         = <PUBLIC_IP>
lbc_keycloak_role_arn     = arn:aws:iam::...
lbc_stigman_role_arn      = arn:aws:iam::...
ebs_csi_role_arn          = arn:aws:iam::...
```

### 2. Configure kubectl contexts

```bash
aws eks update-kubeconfig --region us-east-2 --name keycloak-eks --alias keycloak
aws eks update-kubeconfig --region us-east-2 --name stigman-eks  --alias stigman
```

### 3. Install the AWS Load Balancer Controller (both clusters)

Run the block below for **keycloak**, then repeat substituting `--kube-context stigman` and the `lbc_stigman_role_arn`.

```bash
# Add the EKS Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Keycloak cluster
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --kube-context keycloak \
  --set clusterName=keycloak-eks \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<lbc_keycloak_role_arn>

# STIGman cluster
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --kube-context stigman \
  --set clusterName=stigman-eks \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<lbc_stigman_role_arn>
```

### 4. Issue ACM certificates

You need two self-signed (or CA-signed) certs issued in ACM — one per ALB.

```bash
# Using AWS Private CA or import self-signed certs:
aws acm import-certificate \
  --certificate fileb://keycloak.crt \
  --private-key  fileb://keycloak.key \
  --region us-east-2

aws acm import-certificate \
  --certificate fileb://stigman.crt \
  --private-key  fileb://stigman.key \
  --region us-east-2
```

Save the returned ARNs — you'll need them in the next step.

### 5. Update cert ARNs in manifests

Edit `k8s/keycloak/keycloak.yaml` and `k8s/stigman/stigman-api.yaml`:

```yaml
alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:us-east-2:<ACCOUNT>:certificate/<ID>"
```

### 6. Deploy Keycloak

```bash
kubectl apply -f k8s/keycloak/keycloak.yaml --context keycloak
```

Wait for the ALB to provision (~2 min):

```bash
kubectl get ingress -n keycloak --context keycloak
# NAME       CLASS   HOSTS   ADDRESS                                   PORTS
# keycloak   alb     *       internal-k8s-keycloak-...elb.amazonaws.com   80, 443
```

Save the Keycloak ALB DNS name — you need it in the next step.

### 7. Deploy STIGman (DB + API)

First, update `STIGMAN_OIDC_PROVIDER` in `k8s/stigman/stigman-api.yaml` with the Keycloak ALB DNS:

```yaml
- name: STIGMAN_OIDC_PROVIDER
  value: "https://<KEYCLOAK_ALB_DNS>/realms/stigman"
```

Then deploy:

```bash
kubectl apply -f k8s/stigman/stigman-db.yaml  --context stigman
kubectl apply -f k8s/stigman/stigman-api.yaml --context stigman
```

Wait for MySQL to become ready before checking the API:

```bash
kubectl rollout status deployment/mysql     -n stigman --context stigman
kubectl rollout status deployment/stigman-api -n stigman --context stigman
```

Verify the API connected to both Keycloak and MySQL:

```bash
kubectl logs -n stigman --context stigman deployment/stigman-api | \
  grep -E '"success":true|"db":true|"oidc":true'
```

### 8. Fix Keycloak ALB health check

The Keycloak root path returns a `302` redirect, which the ALB health check rejects by default.

1. Go to **EC2 → Target Groups**
2. Find `k8s-keycloak-keycloak-XXXXX`
3. **Health checks → Edit → Success codes**: change to `200,301,302`
4. Save and wait for the target to show **Healthy**

### 9. RDP and browser setup

```bash
# From your Linux VM:
xfreerdp /v:<WINDOWS_PUBLIC_IP> /u:Administrator /p:'<PASSWORD>' \
  /cert:ignore /dynamic-resolution
```

Inside the RDP session, open PowerShell to trust the self-signed certs:

```powershell
# Import STIGman cert
$wr = [Net.WebRequest]::Create("https://<STIGMAN_ALB_DNS>")
$wr.ServerCertificateValidationCallback = {$true}
try { $wr.GetResponse() } catch {}
$c = $wr.ServicePoint.Certificate
[IO.File]::WriteAllBytes("C:\stigman.cer",
  $c.Export([Security.Cryptography.X509Certificates.X509ContentType]::Cert))
certutil -addstore "Root" C:\stigman.cer

# Import Keycloak cert
$wr2 = [Net.WebRequest]::Create("https://<KEYCLOAK_ALB_DNS>")
$wr2.ServerCertificateValidationCallback = {$true}
try { $wr2.GetResponse() } catch {}
$c2 = $wr2.ServicePoint.Certificate
[IO.File]::WriteAllBytes("C:\keycloak.cer",
  $c2.Export([Security.Cryptography.X509Certificates.X509ContentType]::Cert))
certutil -addstore "Root" C:\keycloak.cer

Stop-Process -Name msedge -Force
```

Then in Keycloak admin (`https://<KEYCLOAK_ALB_DNS>/admin`, login: `admin` / `password`):

1. Switch realm: master → **stigman**
2. Clients → `stig-manager` → Settings
3. Valid redirect URIs: `*`
4. Web origins: `*`
5. Save

Access STIGman at `https://<STIGMAN_ALB_DNS>` — sign in with `stigmanager` / `stigmanager`.


## Kubernetes Manifests Reference

### keycloak.yaml

| Resource | Detail |
|---|---|
| Namespace | `keycloak` |
| Image | `nuwcdivnpt/stig-manager-auth:latest` |
| Ports | `8080` (HTTP), `8443` (HTTPS) |
| Service type | `ClusterIP` |
| Ingress scheme | `internal` (ALB) |
| TLS termination | At the ALB (ACM cert), backend is plain HTTP |
| Key env vars | `KC_HTTP_ENABLED=true`, `KC_PROXY=edge`, `KC_PROXY_HEADERS=xforwarded` |

`KC_PROXY=edge` tells Keycloak it sits behind a TLS-terminating reverse proxy. `KC_HOSTNAME_STRICT=false` allows access via the ALB DNS name without a fixed hostname.

### stigman-db.yaml

| Resource | Detail |
|---|---|
| Image | `mysql:8` |
| Port | `3306` |
| Root password | `rootpw` |
| App user/pass | `stigman` / `stigman` |
| Schema | `stigman` |
| Storage | 20Gi `gp2` PVC via EBS CSI driver |
| Tuning args | `--innodb-buffer-pool-size=1024M --sort_buffer_size=16M` |

The MySQL `Service` name is `mysql.stigman.svc.cluster.local` — this is the value used by `STIGMAN_DB_HOST` in the API deployment.

### stigman-api.yaml

| Resource | Detail |
|---|---|
| Image | `nuwcdivnpt/stig-manager:latest` |
| API port | `54000` |
| DB connection | `mysql.stigman.svc.cluster.local:3306` |
| OIDC provider | Keycloak ALB DNS + `/realms/stigman` |
| TLS | `NODE_TLS_REJECT_UNAUTHORIZED=0` (self-signed cert bypass) |
| Ingress | Internal ALB, SSL redirect to :443 |

---

## Default Credentials

> **These are lab defaults — change all of them before connecting to any real network.**

| Service | Username | Password |
|---|---|---|
| STIGman — full admin | `stigmanager` | `stigmanager` |
| STIGman — collection manager | `collection` | `collection` |
| STIGman — basic user | `lvl1` | `lvl1` |
| Keycloak admin console | `admin` | `password` |
| MySQL root | `root` | `rootpw` |
| MySQL app user | `stigman` | `stigman` |
| Windows jump box | `Administrator` | (decrypt from AWS console) |

---

## Teardown

Run in order to avoid dependency errors. See `scripts/teardown.sh` for the automated version.

```bash
# 1. Delete ALBs (created by LBC, not tracked by Terraform)
aws elbv2 describe-load-balancers --region us-east-2 \
  --query "LoadBalancers[?contains(LoadBalancerName,'k8s')].LoadBalancerArn" \
  --output text | xargs -n1 aws elbv2 delete-load-balancer --region us-east-2 --load-balancer-arn

sleep 30

# 2. Delete orphaned target groups
aws elbv2 describe-target-groups --region us-east-2 \
  --query "TargetGroups[?contains(TargetGroupName,'k8s')].TargetGroupArn" \
  --output text | xargs -n1 aws elbv2 delete-target-group --region us-east-2 --target-group-arn

# 3. Delete EBS volumes tagged to the cluster
aws ec2 describe-volumes --region us-east-2 \
  --filters "Name=tag-key,Values=kubernetes.io/cluster/stigman-eks" \
  --query "Volumes[].VolumeId" --output text | \
  xargs -n1 aws ec2 delete-volume --region us-east-2 --volume-id

# 4. Destroy everything Terraform manages
cd terraform/
terraform destroy
```

---




## References

- [STIG Manager GitHub](https://github.com/NUWCDIVNPT/stig-manager)
- [stig-manager-auth (Keycloak image)](https://hub.docker.com/r/nuwcdivnpt/stig-manager-auth)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- [terraform-aws-modules/eks](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
