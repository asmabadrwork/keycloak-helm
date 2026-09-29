# Keycloak 26.x Deployment Guide: New EKS Cluster & New RDS Database

This guide provides an end-to-end, step-by-step blueprint to deploy **Keycloak 26.x (Quarkus distribution)** on a **brand-new AWS EKS cluster** and **new Amazon RDS PostgreSQL database** with identical node pool taints (`dedicated=application:NoSchedule`).

---

## Targeted Environment Profile
* **New EKS Cluster Endpoint:** `https://4535B45FEA4F6FE4ACB0BDB9DFB1B7F9.gr7.ap-south-1.eks.amazonaws.com`
* **Cluster OIDC ID:** `4535B45FEA4F6FE4ACB0BDB9DFB1B7F9`
* **AWS Region:** `ap-south-1`
* **Target Namespace:** `keycloak`
* **Node Taints:** `dedicated=application:NoSchedule` and `dedicated=database:NoSchedule`

---

## Comprehensive Summary of All Changes Needed

| Component | What Needs to Change | Location / Tool |
| :--- | :--- | :--- |
| **AWS IAM Role Trust Policy** | Update OIDC Provider ID to `4535B45FEA4F6FE4ACB0BDB9DFB1B7F9` | AWS IAM Console / CLI |
| **AWS IAM Policy** | Ensure `secretsmanager:GetSecretValue` and `kms:Decrypt` (for new RDS KMS key) are allowed | AWS IAM Console |
| **RDS Security Group** | Allow TCP Port `5432` from NEW EKS Cluster VPC CIDR | AWS VPC / EC2 Security Groups |
| **RDS PostgreSQL DB** | Create database `keycloak` on the new RDS instance | `kubectl run` temporary Postgres pod |
| **Secrets Manager** | Create/Verify secrets for DB password and admin password | AWS Secrets Manager |
| **External Secrets Operator** | Install ESO on NEW EKS cluster with CRDs & Webhook Tolerations | Helm CLI |
| **`chart/values.yaml`** | Update `database.host`, `externalSecrets.awsSecretName`, `serviceAccount.annotations` (IAM Role ARN), `domain`/`hostname` | Helm values file |

---

## Step-by-Step Implementation Guide

### STEP 1: Configure `kubectl` for the NEW EKS Cluster
Connect your terminal to the new EKS cluster:
```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name <NEW_EKS_CLUSTER_NAME>
```
Verify context connection:
```bash
kubectl cluster-info
# Should show endpoint: https://4535B45FEA4F6FE4ACB0BDB9DFB1B7F9.gr7.ap-south-1.eks.amazonaws.com
```

---

### STEP 2: Configure AWS IAM OIDC & IRSA Role

#### 2.1 Retrieve the OIDC Issuer URL for the New Cluster
Verify the OIDC ID matches `4535B45FEA4F6FE4ACB0BDB9DFB1B7F9`:
```bash
aws eks describe-cluster --name <NEW_EKS_CLUSTER_NAME> --region ap-south-1 --query "cluster.identity.oidc.issuer" --output text
```
*Expected Output:* `https://oidc.eks.ap-south-1.amazonaws.com/id/4535B45FEA4F6FE4ACB0BDB9DFB1B7F9`

#### 2.2 Create IAM OIDC Identity Provider (If Not Already Enabled)
```bash
eksctl utils associate-iam-oidc-provider --cluster <NEW_EKS_CLUSTER_NAME> --region ap-south-1 --approve
```

#### 2.3 Create/Update IAM Trust Policy (`trust-policy.json`)
Create a trust policy file for the IAM Role referencing the NEW EKS OIDC ID:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/4535B45FEA4F6FE4ACB0BDB9DFB1B7F9"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/4535B45FEA4F6FE4ACB0BDB9DFB1B7F9:sub": "system:serviceaccount:keycloak:keycloak-service-account",
          "oidc.eks.ap-south-1.amazonaws.com/id/4535B45FEA4F6FE4ACB0BDB9DFB1B7F9:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

#### 2.4 Create / Update the IAM Role & Attach Policy
```bash
# Update trust relationship on existing role or create new role
aws iam update-assume-role-policy \
  --role-name KeycloakSecretsManagerRole \
  --policy-document file://trust-policy.json

# Ensure KeycloakSecretsManagerPolicy is attached
aws iam attach-role-policy \
  --role-name KeycloakSecretsManagerRole \
  --policy-arn arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:policy/KeycloakSecretsManagerPolicy
```

> [!IMPORTANT]
> **Policy Check:** Make sure `KeycloakSecretsManagerPolicy` includes `"kms:Decrypt"` if your new AWS Secrets Manager secret is encrypted with a custom KMS key!

---

### STEP 3: RDS PostgreSQL & Security Group Setup

#### 3.1 Update RDS Security Group Inbound Rule
1. Open AWS EC2 / VPC Security Groups.
2. Select the Security Group attached to your **NEW RDS PostgreSQL Instance**.
3. Add Inbound Rule:
   * **Type:** PostgreSQL (TCP)
   * **Port:** `5432`
   * **Source:** NEW EKS Cluster VPC CIDR (e.g. `10.10.0.0/16` or Security Group ID of worker nodes).

#### 3.2 Create `keycloak` Database in New RDS
Run a temporary pod to create the database:
```bash
kubectl run pg-init --rm -it --image=postgres:15-alpine --restart=Never -- \
  psql -h <NEW_RDS_ENDPOINT_FQDN> -U <MASTER_USERNAME> -d postgres -c "CREATE DATABASE keycloak;"
```

---

### STEP 4: Install External Secrets Operator (ESO) on New Cluster

Deploy ESO with CRDs enabled and webhook node tolerations for your tainted worker nodes:
```bash
# 1. Add ESO Helm Repository
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# 2. Install ESO into namespace external-secrets
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true \
  --set webhook.tolerations[0].key=dedicated \
  --set webhook.tolerations[0].operator=Equal \
  --set webhook.tolerations[0].value=application \
  --set webhook.tolerations[0].effect=NoSchedule
```

---

### STEP 5: Code Changes in Helm Chart (`chart/values.yaml`)

Edit [`chart/values.yaml`](file:///c:/Users/lenovo/OneDrive/Desktop/Keycloak/chart/values.yaml) and update the following configuration sections:

#### A. Ingress & Hostname Section
```yaml
domain: "keycloak-aws.opstree.dev"      # Change to your target domain name
hostname: "keycloak-aws.opstree.dev"    # Must match domain FQDN
```

#### B. Database Section
```yaml
database:
  vendor: postgres
  host: "<NEW_RDS_ENDPOINT_FQDN>"       # e.g., new-rds.cl0caskygcz8.ap-south-1.rds.amazonaws.com
  port: 5432
  name: "keycloak"
  username: "<NEW_DB_MASTER_USER>"      # e.g., postgres
  createSecret: false
  existingSecret: "keycloak-db-secret"
```

#### C. External Secrets Section
```yaml
externalSecrets:
  enabled: true
  awsRegion: "ap-south-1"
  awsSecretName: "<NEW_AWS_SECRET_NAME>" # e.g., rds!db-74515dc7-b661-4d00-88f9-89033497296c
  adminPasswordSecretName: "production/keycloak/admin"
  propertyPassword: "password"
  propertyAdminPassword: "admin-password"
```

#### D. ServiceAccount & IRSA Section
```yaml
serviceAccount:
  create: true
  name: keycloak-service-account
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:role/KeycloakSecretsManagerRole"
```

#### E. Tolerations Section (Keep Intact)
```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "application"
    effect: "NoSchedule"
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

---

### STEP 6: Provision TLS Secret & Deploy Keycloak

#### 6.1 Create Namespace & TLS Certificate Secret
```bash
# Create target namespace
kubectl create namespace keycloak

# Upload TLS Certificate Secret
kubectl create secret tls keycloak-tls-secret \
  --cert=/path/to/fullchain.pem \
  --key=/path/to/privkey.key \
  -n keycloak
```

#### 6.2 Deploy Keycloak Helm Release
```bash
helm upgrade --install keycloak ./chart \
  --namespace keycloak \
  --create-namespace
```

---

### STEP 7: Post-Deployment Verification

Run the following commands to verify a successful deployment:

#### 1. Verify ExternalSecret Sync Status
```bash
kubectl get externalsecret,secretstore,secret -n keycloak
```
*Verification Goal:* `STATUS: SecretSynced`, `READY: True`. Secret `keycloak-db-secret` created.

#### 2. Check Pod Scheduling & Status
```bash
kubectl get pods -n keycloak -o wide
```
*Verification Goal:* 2 Pods in `Running` status on nodes with taint `dedicated=application:NoSchedule`.

#### 3. Inspect Live Startup Logs
```bash
kubectl logs -f deployment/keycloak -n keycloak
```
*Verification Goal:* Look for `Keycloak 26.3.3 started in ...ms`. Zero DB connection errors.

#### 4. Check Ingress Controller Endpoint
```bash
kubectl get ingress -n keycloak
```
*Verification Goal:* Ingress host matches `keycloak-aws.opstree.dev` with TLS termination enabled.

---

## Quick Reference Summary of Checklist

- [ ] Connected `kubectl` to new EKS cluster endpoint `4535B45FEA4F6FE4ACB0BDB9DFB1B7F9`.
- [ ] Updated IAM Role trust relationship for OIDC provider `4535B45FEA4F6FE4ACB0BDB9DFB1B7F9`.
- [ ] Added PostgreSQL 5432 inbound rule to new RDS Security Group.
- [ ] Executed `CREATE DATABASE keycloak;` on new RDS.
- [ ] Installed External Secrets Operator with `--set installCRDs=true` and webhook tolerations.
- [ ] Updated `chart/values.yaml` (`database.host`, `externalSecrets.awsSecretName`, `serviceAccount.annotations`).
- [ ] Created TLS secret `keycloak-tls-secret` in namespace `keycloak`.
- [ ] Executed `helm upgrade --install keycloak ./chart -n keycloak`.
- [ ] Confirmed pods `Running` and `SecretSynced = True`.
