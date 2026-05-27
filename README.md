# Enterprise DevSecOps CI/CD Platform — Full Implementation Guide

> **3-Tier Node.js + MySQL application** deployed on AWS EKS with full DevSecOps pipeline, GitOps-based QA → PROD promotion, secrets management via AWS Secrets Manager, observability via Prometheus + Loki + Grafana, and OIDC-based keyless authentication.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Technology Stack & Tool Purposes](#3-technology-stack--tool-purposes)
4. [Repository Structure](#4-repository-structure)
5. [What to Change for Your Setup](#5-what-to-change-for-your-setup)
6. [Phase 1 — Local Tools Setup (Windows)](#phase-1--local-tools-setup-windows)
7. [Phase 2 — AWS Account & IAM Setup](#phase-2--aws-account--iam-setup)
8. [Phase 3 — EKS Cluster Setup](#phase-3--eks-cluster-setup)
9. [Phase 4 — AWS ALB Ingress Controller](#phase-4--aws-alb-ingress-controller)
10. [Phase 5 — EBS CSI Driver & Storage](#phase-5--ebs-csi-driver--storage)
11. [Phase 6 — GitHub OIDC Auth for QA](#phase-6--github-oidc-auth-for-qa)
12. [Phase 7 — GitHub OIDC Auth for PROD](#phase-7--github-oidc-auth-for-prod)
13. [Phase 8 — AWS Secrets Manager + ESO (QA)](#phase-8--aws-secrets-manager--eso-qa)
14. [Phase 9 — AWS Secrets Manager + ESO (PROD)](#phase-9--aws-secrets-manager--eso-prod)
15. [Phase 10 — MySQL StatefulSet Deployment](#phase-10--mysql-statefulset-deployment)
16. [Phase 11 — Node.js App Deployment & Ingress](#phase-11--nodejs-app-deployment--ingress)
17. [Phase 12 — GitHub Actions CI/CD Pipeline](#phase-12--github-actions-cicd-pipeline)
18. [Phase 13 — Observability Stack (Prometheus + Loki + Grafana)](#phase-13--observability-stack-prometheus--loki--grafana)
19. [Verification Checklist](#verification-checklist)
20. [Common Errors & Fixes](#common-errors--fixes)

---

## 1. Project Overview

This is a full-stack **User Management Platform** — a React frontend + Node.js/Express backend + MySQL database — deployed on Amazon EKS with enterprise-grade DevSecOps practices.

**Application:** Users can be created, read, updated, and deleted via a React UI that calls REST APIs on the Node.js server, which persists data to MySQL.

**Two environments run in the same EKS cluster in separate namespaces:**
- `qa` — deployed automatically on every push to the `qa` branch
- `prod` — deployed by promoting the QA image via a push to `main`

**Key design decisions:**
- No long-lived AWS credentials anywhere — GitHub Actions uses OIDC short-lived tokens
- No hardcoded secrets anywhere — all credentials live in AWS Secrets Manager and are synced into Kubernetes by External Secrets Operator
- All security scanning (Gitleaks, Checkov, Trivy) runs in the pipeline before any image is pushed
- Prod gets the exact same image as QA — just retagged — ensuring what was tested is what ships

---

## 2. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            DEVELOPER WORKFLOW                                    │
│                                                                                  │
│   git push → qa branch          git push / PR → main branch                     │
│         │                                  │                                     │
│         ▼                                  ▼                                     │
│  ┌─────────────────────┐      ┌────────────────────────┐                        │
│  │  QA CI/CD Pipeline  │      │   PROD CD Pipeline     │                        │
│  │  (qa-cicd.yaml)     │      │   (prod-cd.yaml)       │                        │
│  │                     │      │                        │                        │
│  │ 1. Gitleaks         │      │ 1. Pull QA :latest     │                        │
│  │ 2. Checkov (k8s,    │      │ 2. Retag → prod-{sha}  │                        │
│  │    tf, dockerfile)  │      │ 3. Push to DockerHub   │                        │
│  │ 3. Trivy FS scan    │      │ 4. Update prod manifest│                        │
│  │ 4. Lint + Test      │      │ 5. Deploy to prod ns   │                        │
│  │ 5. Build client     │      └────────────────────────┘                        │
│  │ 6. Docker build     │                  │                                     │
│  │ 7. Trivy img scan   │                  │ GitHub OIDC                         │
│  │ 8. SBOM generation  │                  │ (AWS_ROLE_TO_ASSUME1)               │
│  │ 9. Push to DockerHub│                  ▼                                     │
│  │10. Update manifest  │      ┌────────────────────────┐                        │
│  │11. Deploy to QA     │      │  GitHubActionsEKS      │                        │
│  └─────────────────────┘      │  DeployRolePROD        │                        │
│         │                     └────────────────────────┘                        │
│         │ GitHub OIDC                                                            │
│         │ (AWS_ROLE_TO_ASSUME)                                                  │
│         ▼                                                                        │
│  ┌──────────────────────┐                                                        │
│  │ GitHubActionsEKS     │                                                        │
│  │ DeployRoleQA         │                                                        │
│  └──────────────────────┘                                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
                    │                              │
                    ▼                              ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                        AWS EKS CLUSTER (my-cluster, ap-south-1)               │
│                                                                               │
│  ┌─────────────────────────────────┐  ┌──────────────────────────────────┐   │
│  │         NAMESPACE: qa           │  │         NAMESPACE: prod          │   │
│  │                                 │  │                                  │   │
│  │  ┌─────────────┐                │  │  ┌─────────────┐                 │   │
│  │  │ nodejs-app  │◄── regcred     │  │  │ nodejs-app  │◄── regcred      │   │
│  │  │ Deployment  │    (DockerHub) │  │  │ Deployment  │    (DockerHub)  │   │
│  │  └──────┬──────┘                │  │  └──────┬──────┘                 │   │
│  │         │ env from              │  │         │ env from               │   │
│  │         ▼ mysql-secret          │  │         ▼ mysql-secret           │   │
│  │  ┌─────────────┐                │  │  ┌─────────────┐                 │   │
│  │  │   MySQL     │                │  │  │   MySQL     │                 │   │
│  │  │ StatefulSet │                │  │  │ StatefulSet │                 │   │
│  │  │ + EBS 5Gi   │                │  │  │ + EBS 5Gi   │                 │   │
│  │  └─────────────┘                │  │  └─────────────┘                 │   │
│  │                                 │  │                                  │   │
│  │  ┌─────────────┐                │  │  ┌─────────────┐                 │   │
│  │  │ExternalSecret│               │  │  │ExternalSecret│                │   │
│  │  │mysql-external│               │  │  │mysql-external│                │   │
│  │  │-secret       │               │  │  │-secret       │                │   │
│  │  └──────┬───────┘               │  │  └──────┬───────┘                │   │
│  │         │ syncs                 │  │         │ syncs                  │   │
│  │         ▼                       │  │         ▼                        │   │
│  │  ┌─────────────┐                │  │  ┌─────────────┐                 │   │
│  │  │ mysql-secret│                │  │  │ mysql-secret│                 │   │
│  │  │ (K8s Secret)│                │  │  │ (K8s Secret)│                 │   │
│  │  └─────────────┘                │  │  └─────────────┘                 │   │
│  │                                 │  │                                  │   │
│  │  ┌─────────────┐                │  │  ┌─────────────┐                 │   │
│  │  │SecretStore  │                │  │  │SecretStore  │                 │   │
│  │  │eso-qa-sa    │                │  │  │eso-prod-sa  │                 │   │
│  │  │(IRSA role)  │                │  │  │(IRSA role)  │                 │   │
│  │  └─────────────┘                │  │  └─────────────┘                 │   │
│  └─────────────────────────────────┘  └──────────────────────────────────┘   │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐   │
│  │                    NAMESPACE: monitoring                               │   │
│  │                                                                        │   │
│  │  Prometheus (kube-prometheus-stack) → scrapes qa namespace pods        │   │
│  │  Loki (SingleBinary) → receives logs from Grafana Alloy → S3 storage  │   │
│  │  Grafana Alloy (DaemonSet) → collects logs + metrics from qa pods     │   │
│  │  Grafana → dashboards, pre-built from grafana.com, exposed via ALB    │   │
│  └────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐   │
│  │                    NAMESPACE: kube-system                              │   │
│  │                                                                        │   │
│  │  AWS Load Balancer Controller → provisions ALB for all Ingress objs   │   │
│  │  EBS CSI Driver → provisions EBS volumes for StatefulSets             │   │
│  │  External Secrets Operator → syncs AWS Secrets → K8s Secrets          │   │
│  └────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌─── ALB (internet-facing) ─────────────────────────────────────────────┐   │
│  │  jp2op-project.site → qa/nodejs-service:5000                          │   │
│  │  jp2op-project.site → prod/nodejs-service:5000                        │   │
│  │  ALB → Grafana (monitoring ns)                                        │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────────┘
                    │                              │
         ┌──────────┘                              └──────────┐
         ▼                                                    ▼
┌─────────────────────┐                        ┌─────────────────────────┐
│  AWS Secrets Manager│                        │       Amazon S3         │
│                     │                        │                         │
│  qa/mysql-secret    │                        │  qa-demo-s3-777         │
│  prod/mysql-secret  │                        │  (Loki log storage)     │
└─────────────────────┘                        └─────────────────────────┘
```

---

## 3. Technology Stack & Tool Purposes

| Tool | Purpose |
|------|---------|
| **Node.js / Express** | Backend REST API server handling CRUD operations for users |
| **React + Webpack** | Frontend SPA bundled and served as static files by the Node.js server |
| **MySQL 8.0** | Relational database storing users table with id, name, email, role |
| **Docker** | Packages the entire app (client + server) into a single image |
| **Amazon EKS** | Managed Kubernetes cluster that runs all workloads |
| **eksctl** | CLI tool to create and manage EKS clusters and IRSA service accounts |
| **kubectl** | CLI to apply Kubernetes manifests and inspect cluster state |
| **Helm** | Package manager for installing complex Kubernetes components |
| **AWS ALB Ingress Controller** | Provisions AWS Application Load Balancers from Kubernetes Ingress objects |
| **AWS EBS CSI Driver** | Bridges Kubernetes PersistentVolumeClaims to AWS EBS volumes |
| **External Secrets Operator (ESO)** | Syncs secrets from AWS Secrets Manager into Kubernetes Secrets automatically |
| **AWS Secrets Manager** | Secure storage for database credentials — single source of truth |
| **IRSA (IAM Roles for Service Accounts)** | Lets Kubernetes pods assume AWS IAM roles without hardcoded credentials |
| **GitHub Actions** | CI/CD automation platform that runs the pipeline on push events |
| **GitHub OIDC** | Keyless authentication — GitHub Actions assumes AWS IAM roles without static keys |
| **Gitleaks** | Scans git history and source code for accidentally committed secrets |
| **Checkov** | Static analysis of Terraform, Kubernetes YAML, and Dockerfile for misconfigurations |
| **Trivy** | Vulnerability scanner for filesystem dependencies and container images |
| **Anchore SBOM** | Generates Software Bill of Materials for source code and Docker images |
| **Prometheus** | Collects and stores time-series metrics from cluster nodes and pods |
| **Grafana Loki** | Log aggregation system — stores logs from pods in S3 |
| **Grafana Alloy** | Agent running as DaemonSet that ships logs and metrics to Loki/Prometheus |
| **Grafana** | Visualization platform for metrics, logs, and dashboards |
| **AWS S3** | Object storage used by Loki for long-term log retention |
| **StorageClass (ebs-sc)** | Kubernetes object defining how EBS gp3 volumes are provisioned on demand |

---

## 4. Repository Structure

```
user-platform-project/
├── .github/
│   └── workflows/
│       ├── qa-cicd.yaml          # Full CI + CD pipeline triggered on qa branch push
│       └── prod-cd.yaml          # CD-only pipeline triggered on main branch push
├── client/                       # React frontend
│   ├── src/
│   │   ├── App.js                # Root React component
│   │   ├── api/users.js          # Axios API calls to /api/users
│   │   └── components/
│   │       ├── UsersList.js      # Lists all users
│   │       └── UserItem.js       # Single user row with edit/delete
│   └── webpack.config.js         # Bundles React into client/public/bundle.js
├── server/                       # Node.js/Express backend
│   ├── server.js                 # Entry point — connects to MySQL, defines routes, serves static
│   ├── app.js                    # Express app factory
│   ├── config/db.js              # MySQL connection using env vars (DB_HOST, DB_USER, etc.)
│   ├── controllers/userController.js  # CRUD logic
│   ├── models/userModel.js       # SQL query wrappers
│   └── routes/userRoutes.js      # GET/POST/PUT/DELETE /api/users
├── k8-manifests/
│   ├── qa/
│   │   ├── app-deployment.yaml   # nodejs-app Deployment in qa namespace
│   │   ├── app-svc.yaml          # ClusterIP Service exposing port 5000
│   │   └── app-ingress.yaml      # ALB Ingress for jp2op-project.site
│   └── prod/
│       ├── app-deployment.yaml   # nodejs-app Deployment in prod namespace
│       ├── app-svc.yaml          # ClusterIP Service exposing port 5000
│       └── app-ingress.yaml      # ALB Ingress for jp2op-project.site
├── My-sql-k8s/                   # MySQL Kubernetes manifests (apply manually)
│   ├── Storage-Class.yaml        # ebs-sc StorageClass using ebs.csi.aws.com
│   ├── secretstore.yaml          # SecretStore — tells ESO how to connect to AWS
│   ├── external-secret.yaml      # ExternalSecret — what to fetch from Secrets Manager
│   ├── mysql-statefulset.yaml    # MySQL StatefulSet with 5Gi EBS volume
│   └── mysql-svc.yaml            # Headless Service for StatefulSet DNS
├── Monitoring/                   # Observability stack (run deploy.ps1)
│   ├── 01-prometheus-values.yaml # kube-prometheus-stack Helm values
│   ├── 02-loki-values.yaml       # Loki Helm values (S3 backend)
│   ├── 03-alloy-values.yaml      # Grafana Alloy DaemonSet config
│   ├── 04-grafana-values.yaml    # Grafana with pre-built dashboards
│   ├── 05-loki-s3-iam-policy.json# IAM policy for Loki to access S3
│   └── deploy.ps1                # One-shot PowerShell deployment script
└── Dockerfile                    # Multi-stage build: React build → Node.js server
```

---

## 5. What to Change for Your Setup

Every time you redo this project, replace these values throughout all files and commands:

| Placeholder | What It Is | Where to Change |
|-------------|-----------|-----------------|
| `796197769514` | Your AWS Account ID | Every IAM ARN, policy, trust policy |
| `my-cluster` | EKS cluster name | eksctl, aws eks, kubectl commands |
| `ap-south-1` | AWS region | All AWS CLI commands and YAML files |
| `jayyp2op` | Your DockerHub username | Dockerfile tags, workflow files, deployment YAMLs |
| `Jp2op/user-platform-project` | Your GitHub repo (owner/name) | OIDC trust policy `sub` condition |
| `jp2op-project.site` | Your domain name | app-ingress.yaml in qa and prod |
| `arn:aws:acm:...:certificate/...` | Your ACM certificate ARN | app-ingress.yaml in qa and prod |
| `qa-demo-s3-777` | Your S3 bucket name for Loki | deploy.ps1 `$S3_BUCKET` variable |
| `eksctl-my-cluster-nodegroup-...-NodeInstanceRole-...` | Your EKS node IAM role name | deploy.ps1 `$NODE_ROLE_NAME` variable |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin password | deploy.ps1 `$GRAFANA_ADMIN_PASSWORD` variable |

> **Find your node role name:** `aws iam list-roles --query "Roles[?contains(RoleName,'NodeInstanceRole')].RoleName" --output table`

---

## Phase 1 — Local Tools Setup (Windows) 

### Step 1.1 — Create tools folder (Optional)
```powershell
mkdir C:\DevOpsTools
```

### Step 1.2 — Install AWS CLI
```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
aws --version
# Expected: aws-cli/2.x.x
```

### Step 1.3 — Install kubectl
```powershell
curl.exe -LO https://dl.k8s.io/release/v1.33.0/bin/windows/amd64/kubectl.exe
move kubectl.exe C:\DevOpsTools\
C:\DevOpsTools\kubectl.exe version --client
```

### Step 1.4 — Install Helm
```powershell
curl.exe -LO https://get.helm.sh/helm-v3.15.0-windows-amd64.zip
tar -xf helm-v3.15.0-windows-amd64.zip
move windows-amd64\helm.exe C:\DevOpsTools\
```

### Step 1.5 — Install eksctl
```powershell
curl.exe -LO https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Windows_amd64.zip
tar -xf eksctl_Windows_amd64.zip
move eksctl.exe C:\DevOpsTools\
```

### Step 1.6 — Add to PATH (run VS Code as Administrator)
```powershell
[Environment]::SetEnvironmentVariable(
  "Path",
  $env:Path + ";C:\DevOpsTools",
  [EnvironmentVariableTarget]::Machine
)
```

### Step 1.7 — Verify all tools
```powershell
aws --version
kubectl version --client
helm version
eksctl version
```

---

## Phase 2 — AWS Account & IAM Setup

### Step 2.1 — Create IAM Group and User (AWS Console)

1. Open AWS Console → IAM → User groups → Create group
   - Name: `admins`
   - Attach policy: `AdministratorAccess`
2. IAM → Users → Create user
   - Name: `devops-user`
   - Add to group: `admins`
3. Open the user → Security credentials → Create access key
   - Choose: Command Line Interface (CLI)
   - Download the CSV

### Step 2.2 — Configure AWS CLI
```powershell
aws configure
# AWS Access Key ID: <paste from CSV>
# AWS Secret Access Key: <paste from CSV>
# Default region name: ap-south-1
# Default output format: json
```

### Step 2.3 — Verify
```powershell
aws sts get-caller-identity
# Should return your Account ID: 796197769514
```

---

## Phase 3 — EKS Cluster Setup

### Step 3.1 — Create the cluster
```powershell
eksctl create cluster `
  --name my-cluster `
  --region ap-south-1 `
  --nodegroup-name my-nodes `
  --node-type t3.medium `
  --nodes 2 `
  --nodes-min 1 `
  --nodes-max 3 `
  --managed
```
> This takes 15-20 minutes. eksctl creates the VPC, subnets, node group, and kubeconfig automatically.

### Step 3.2 — Verify cluster
```powershell
kubectl get nodes
kubectl get pods -A
```

### Step 3.3 — Check auth mode (must be API_AND_CONFIG_MAP)
```powershell
aws eks describe-cluster `
  --name my-cluster `
  --region ap-south-1 `
  --query "{status:cluster.status,authMode:cluster.accessConfig.authenticationMode}" `
  --output json
```

If it shows `CONFIG_MAP`, update it:
```powershell
aws eks update-cluster-config `
  --name my-cluster `
  --region ap-south-1 `
  --access-config authenticationMode=API_AND_CONFIG_MAP
```

### Step 3.4 — Associate OIDC provider (for IRSA)
```powershell
eksctl utils associate-iam-oidc-provider `
  --cluster my-cluster `
  --region ap-south-1 `
  --approve
```

### Step 3.5 — Create namespaces
```powershell
kubectl create namespace qa
kubectl create namespace prod
kubectl get namespaces
```

---

## Phase 4 — AWS ALB Ingress Controller

The ALB controller watches Kubernetes Ingress objects and automatically provisions AWS Application Load Balancers.

### Step 4.1 — Tag your public subnets for ALB discovery
```powershell
aws ec2 describe-subnets `
  --region ap-south-1 `
  --filters "Name=tag:kubernetes.io/role/elb,Values=1" `
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone}" `
  --output table
```
If no subnets are returned, tag them in the AWS Console:
- Key: `kubernetes.io/role/elb` → Value: `1`
- Key: `kubernetes.io/cluster/my-cluster` → Value: `shared`

### Step 4.2 — Download the IAM policy
```powershell
Invoke-WebRequest `
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json `
  -OutFile iam_policy.json
```

### Step 4.3 — Create the IAM policy
```powershell
aws iam create-policy `
  --policy-name AWSLoadBalancerControllerIAMPolicy `
  --policy-document file://iam_policy.json
```

### Step 4.4 — Create IRSA service account
> **Change:** Replace `796197769514` with your account ID
```powershell
eksctl create iamserviceaccount `
  --cluster=my-cluster `
  --namespace=kube-system `
  --name=aws-load-balancer-controller `
  --attach-policy-arn=arn:aws:iam::796197769514:policy/AWSLoadBalancerControllerIAMPolicy `
  --override-existing-serviceaccounts `
  --region ap-south-1 `
  --approve
```

### Step 4.5 — Install via Helm
```powershell
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=my-cluster `
  --set serviceAccount.create=false `
  --set serviceAccount.name=aws-load-balancer-controller `
  --version 1.14.0
```

### Step 4.6 — Verify
```powershell
kubectl get deployment -n kube-system aws-load-balancer-controller
# AVAILABLE must be 1
```

---

## Phase 5 — EBS CSI Driver & Storage

The EBS CSI driver lets Kubernetes create and attach AWS EBS volumes for StatefulSets like MySQL.

### Step 5.1 — Create IAM role for EBS CSI
```powershell
eksctl create iamserviceaccount `
  --name ebs-csi-controller-sa `
  --namespace kube-system `
  --cluster my-cluster `
  --region ap-south-1 `
  --role-name AmazonEKS_EBS_CSI_DriverRole `
  --role-only `
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy `
  --approve
```

### Step 5.2 — Install EBS CSI addon
> **Change:** Replace `796197769514` with your account ID
```powershell
aws eks create-addon `
  --cluster-name my-cluster `
  --addon-name aws-ebs-csi-driver `
  --region ap-south-1 `
  --service-account-role-arn arn:aws:iam::796197769514:role/AmazonEKS_EBS_CSI_DriverRole `
  --resolve-conflicts OVERWRITE
```

If addon already exists, use `update-addon` instead of `create-addon`.

### Step 5.3 — Verify EBS pods
```powershell
kubectl get pods -n kube-system | findstr ebs
# Should see ebs-csi-controller and ebs-csi-node pods Running
```

### Step 5.4 — Apply StorageClass
Save as `storageclass-ebs.yaml`:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  fsType: ext4
```
```powershell
kubectl apply -f storageclass-ebs.yaml
kubectl get storageclass ebs-sc
```

> `WaitForFirstConsumer` means the EBS volume is created in the same AZ as the pod — critical for EKS.

---

## Phase 6 — GitHub OIDC Auth for QA

OIDC lets GitHub Actions assume an AWS IAM role using a short-lived token — no static AWS keys needed in GitHub secrets.

### Step 6.1 — Check if OIDC provider already exists
```powershell
aws iam list-open-id-connect-providers `
  --query "OpenIDConnectProviderList[?contains(Arn, 'token.actions.githubusercontent.com')].Arn" `
  --output text
```

If empty, create it:
```powershell
aws iam create-open-id-connect-provider `
  --url https://token.actions.githubusercontent.com `
  --client-id-list sts.amazonaws.com `
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

### Step 6.2 — Create trust policy
> **Change:** Replace `796197769514` with your account ID and `Jp2op/user-platform-project` with your GitHub `owner/repo`
```powershell
$policy = '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Federated":"arn:aws:iam::796197769514:oidc-provider/token.actions.githubusercontent.com"},"Action":"sts:AssumeRoleWithWebIdentity","Condition":{"StringEquals":{"token.actions.githubusercontent.com:aud":"sts.amazonaws.com"},"StringLike":{"token.actions.githubusercontent.com:sub":"repo:Jp2op/user-platform-project:*"}}}]}'

$policy | Out-File -FilePath .\github-oidc-trust-qa.json -Encoding ascii -NoNewline
Get-Content .\github-oidc-trust-qa.json
```

### Step 6.3 — Create QA IAM role
```powershell
aws iam create-role `
  --role-name GitHubActionsEKSDeployRoleQA `
  --assume-role-policy-document file://github-oidc-trust-qa.json
```

### Step 6.4 — Create EKS describe policy
> **Change:** Replace `796197769514` with your account ID
```powershell
$pol = '{"Version":"2012-10-17","Statement":[{"Sid":"EKSDescribeCluster","Effect":"Allow","Action":["eks:DescribeCluster"],"Resource":"arn:aws:eks:ap-south-1:796197769514:cluster/my-cluster"}]}'
$pol | Out-File -FilePath .\eks-describe-cluster-policy.json -Encoding ascii -NoNewline

aws iam create-policy `
  --policy-name GitHubActionsEKSDescribeClusterPolicy `
  --policy-document file://eks-describe-cluster-policy.json
```

### Step 6.5 — Attach policy to QA role
```powershell
aws iam attach-role-policy `
  --role-name GitHubActionsEKSDeployRoleQA `
  --policy-arn arn:aws:iam::796197769514:policy/GitHubActionsEKSDescribeClusterPolicy
```

### Step 6.6 — Create EKS access entry
```powershell
aws eks create-access-entry `
  --cluster-name my-cluster `
  --region ap-south-1 `
  --principal-arn arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRoleQA
```

### Step 6.7 — Associate namespace-scoped access policy
```powershell
aws eks associate-access-policy `
  --cluster-name my-cluster `
  --region ap-south-1 `
  --principal-arn arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRoleQA `
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy `
  --access-scope type=namespace,namespaces=qa
```

### Step 6.8 — Add GitHub repo secret
Go to your repo → **Settings → Secrets and variables → Actions → New repository secret**:
- Name: `AWS_ROLE_TO_ASSUME`
- Value: `arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRoleQA`

### Step 6.9 — Create Docker registry secret in QA
```powershell
kubectl create secret docker-registry regcred `
  --docker-username=<your-dockerhub-username> `
  --docker-password=<your-dockerhub-token> `
  --docker-email=<your-email> `
  -n qa
```

---

## Phase 7 — GitHub OIDC Auth for PROD

Same process as QA but for the `prod` namespace and `main` branch.

### Step 7.1 — Create trust policy
> **Change:** Same replacements as Phase 6
```powershell
$policy = '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Federated":"arn:aws:iam::796197769514:oidc-provider/token.actions.githubusercontent.com"},"Action":"sts:AssumeRoleWithWebIdentity","Condition":{"StringEquals":{"token.actions.githubusercontent.com:aud":"sts.amazonaws.com"},"StringLike":{"token.actions.githubusercontent.com:sub":"repo:Jp2op/user-platform-project:*"}}}]}'

$policy | Out-File -FilePath .\github-oidc-trust-prod.json -Encoding ascii -NoNewline
```

### Step 7.2 — Create PROD IAM role
```powershell
aws iam create-role `
  --role-name GitHubActionsEKSDeployRolePROD `
  --assume-role-policy-document file://github-oidc-trust-prod.json
```

### Step 7.3 — Attach policy (reuse the same describe cluster policy)
```powershell
aws iam attach-role-policy `
  --role-name GitHubActionsEKSDeployRolePROD `
  --policy-arn arn:aws:iam::796197769514:policy/GitHubActionsEKSDescribeClusterPolicy
```

### Step 7.4 — Create EKS access entry
```powershell
aws eks create-access-entry `
  --cluster-name my-cluster `
  --region ap-south-1 `
  --principal-arn arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRolePROD
```

### Step 7.5 — Associate namespace-scoped access policy
```powershell
aws eks associate-access-policy `
  --cluster-name my-cluster `
  --region ap-south-1 `
  --principal-arn arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRolePROD `
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy `
  --access-scope type=namespace,namespaces=prod
```

### Step 7.6 — Add GitHub repo secret
- Name: `AWS_ROLE_TO_ASSUME1`
- Value: `arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRolePROD`

### Step 7.7 — Create Docker registry secret in PROD
```powershell
kubectl create secret docker-registry regcred `
  --docker-username=<your-dockerhub-username> `
  --docker-password=<your-dockerhub-token> `
  --docker-email=<your-email> `
  -n prod
```

---

## Phase 8 — AWS Secrets Manager + ESO (QA)

MySQL credentials are stored in AWS Secrets Manager. External Secrets Operator syncs them into a Kubernetes Secret. MySQL reads that secret. No credentials are ever in YAML files.

### Step 8.1 — Store secret in AWS Secrets Manager
```powershell
$json = @'
{
  "MYSQL_ROOT_PASSWORD": "rootpass",
  "MYSQL_DATABASE": "test_db",
  "MYSQL_USER": "appuser",
  "MYSQL_PASSWORD": "apppass",
  "DATABASE_URL": "mysql://appuser:apppass@mysql:3306/test_db"
}
'@
$utf8NoBom = [System.Text.UTF8Encoding]::new($false)
[System.IO.File]::WriteAllText("$PWD\mysql-secret.json", $json, $utf8NoBom)

aws secretsmanager create-secret `
  --region ap-south-1 `
  --name qa/mysql-secret `
  --description "MySQL credentials for qa namespace" `
  --secret-string file://mysql-secret.json
```

Verify:
```powershell
(aws secretsmanager get-secret-value --region ap-south-1 --secret-id qa/mysql-secret | ConvertFrom-Json).SecretString
```

### Step 8.2 — Install External Secrets Operator
```powershell
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets `
  -n external-secrets `
  --create-namespace

kubectl get pods -n external-secrets
```

### Step 8.3 — Create IAM policy for ESO
> **Change:** Replace `796197769514` with your account ID
```powershell
@"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:796197769514:secret:qa/mysql-secret*"
    }
  ]
}
"@ | Set-Content -Path .\eso-secrets-policy-qa.json

aws iam create-policy `
  --policy-name ESOSecretsManagerQAReadPolicy `
  --policy-document file://eso-secrets-policy-qa.json

$env:POLICY_ARN = aws iam list-policies --scope Local `
  --query "Policies[?PolicyName=='ESOSecretsManagerQAReadPolicy'].Arn" `
  --output text
echo $env:POLICY_ARN
```

### Step 8.4 — Create IRSA service account for ESO
```powershell
eksctl create iamserviceaccount `
  --name eso-qa-sa `
  --namespace qa `
  --cluster my-cluster `
  --region ap-south-1 `
  --role-name eso-qa-secrets-role `
  --attach-policy-arn $env:POLICY_ARN `
  --approve

kubectl get sa eso-qa-sa -n qa -o yaml
# Look for: eks.amazonaws.com/role-arn annotation
```

### Step 8.5 — Apply SecretStore
Save as `My-sql-k8s/secretstore.yaml` (already in repo):
```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: qa
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-qa-sa
```
```powershell
kubectl apply -f My-sql-k8s/secretstore.yaml
kubectl get secretstore -n qa
# READY must be True
```

### Step 8.6 — Apply ExternalSecret
```powershell
kubectl apply -f My-sql-k8s/external-secret.yaml
kubectl get externalsecret -n qa
kubectl get secret mysql-secret -n qa
# mysql-secret must exist
```

---

## Phase 9 — AWS Secrets Manager + ESO (PROD)

Same process as Phase 8 but for the `prod` namespace.

### Step 9.1 — Store PROD secret
```powershell
aws secretsmanager create-secret `
  --region ap-south-1 `
  --name prod/mysql-secret `
  --description "MySQL credentials for prod namespace" `
  --secret-string file://mysql-secret.json
```

### Step 9.2 — Create IAM policy for PROD ESO
> **Change:** Replace `796197769514` with your account ID
```powershell
@"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:796197769514:secret:prod/mysql-secret*"
    }
  ]
}
"@ | Set-Content -Path .\eso-secrets-policy-prod.json

aws iam create-policy `
  --policy-name ESOSecretsManagerPRODReadPolicy `
  --policy-document file://eso-secrets-policy-prod.json

$env:POLICY_ARN_PROD = aws iam list-policies --scope Local `
  --query "Policies[?PolicyName=='ESOSecretsManagerPRODReadPolicy'].Arn" `
  --output text
```

### Step 9.3 — Create IRSA service account for PROD ESO
```powershell
eksctl create iamserviceaccount `
  --name eso-prod-sa `
  --namespace prod `
  --cluster my-cluster `
  --region ap-south-1 `
  --role-name eso-prod-secrets-role `
  --attach-policy-arn $env:POLICY_ARN_PROD `
  --approve
```

### Step 9.4 — Apply SecretStore and ExternalSecret for PROD
The PROD manifests are in `My-sql-k8s/` for the main branch (with `namespace: prod` and `eso-prod-sa`):
```powershell
kubectl apply -f My-sql-k8s/secretstore-prod.yaml
kubectl apply -f My-sql-k8s/external-secret-prod.yaml
kubectl get secret mysql-secret -n prod
```

---

## Phase 10 — MySQL StatefulSet Deployment

### Step 10.1 — Apply MySQL manifests (QA)
```powershell
kubectl apply -f My-sql-k8s/mysql-svc.yaml
kubectl apply -f My-sql-k8s/mysql-statefulset.yaml

kubectl get pods -n qa
# mysql-0 must be Running
kubectl get pvc -n qa
# mysql-storage-mysql-0 must be Bound
```

### Step 10.2 — Apply MySQL manifests (PROD)
```powershell
kubectl apply -f My-sql-k8s/mysql-svc-prod.yaml -n prod
kubectl apply -f My-sql-k8s/mysql-statefulset-prod.yaml -n prod

kubectl get pods -n prod
kubectl get pvc -n prod
```

**How the StatefulSet works:**
- `serviceName: mysql` creates stable DNS `mysql.qa.svc.cluster.local` for the Node.js app to connect to
- `volumeClaimTemplates` automatically creates a PVC that the EBS CSI driver provisions as a 5Gi gp3 EBS volume
- `envFrom: secretRef: mysql-secret` injects all keys from the Kubernetes Secret as environment variables

---

## Phase 11 — Node.js App Deployment & Ingress

The app manifests in `k8-manifests/qa/` and `k8-manifests/prod/` are updated automatically by the CI/CD pipeline with the correct image tag. You only need to apply them manually the first time.

### Step 11.1 — Apply QA app manifests
```powershell
kubectl apply -f k8-manifests/qa/ -n qa
kubectl get pods -n qa
kubectl get svc -n qa
kubectl get ingress -n qa
```

### Step 11.2 — Apply PROD app manifests
```powershell
kubectl apply -f k8-manifests/prod/ -n prod
kubectl get pods -n prod
kubectl get ingress -n prod
```

### Step 11.3 — Get ALB URL
```powershell
kubectl get ingress -n qa
kubectl get ingress -n prod
# Copy the ADDRESS column value — this is your ALB DNS name
```

Point your domain (e.g. `jp2op-project.site`) to the ALB DNS via CNAME in your DNS provider.

**What needs changing in `app-ingress.yaml`:**
- `host:` — change to your domain
- `certificate-arn:` — create a certificate in AWS ACM for your domain and paste the ARN

---

## Phase 12 — GitHub Actions CI/CD Pipeline

### How the QA pipeline works (`qa-cicd.yaml`)

Triggered on push to `qa` branch when `client/`, `server/`, or the workflow file changes.

**Jobs run in this order:**

```
gitleaks-scan
    ├── checkov-terraform ──┐
    ├── checkov-k8s ────────┤
    ├── checkov-dockerfile ─┤──► client-lint ──► client-test ──► client-build ──► docker
    ├── client-trivy-fs ────┘                                                         │
    └── server-trivy-fs ──────────────────────► server-lint ──► server-test ──────────┤
                                                                                       ▼
                                                                               update-qa-manifest
                                                                                       │
                                                                                       ▼
                                                                               deploy-to-k8s (qa)
```

Key stages explained:
- **Gitleaks** — scans entire git history for leaked secrets before anything else runs
- **Checkov** — static analysis of K8s YAMLs, Dockerfile, Terraform (if any) for security misconfigs
- **Trivy FS** — scans `./client` and `./server` directories for vulnerable npm packages
- **Lint + Test** — runs `npm run lint` and `npm test` (non-blocking with `continue-on-error: true`)
- **Client Build** — runs `npm run build` which Webpack bundles React into `client/public/bundle.js`
- **Docker** — builds image, scans it with Trivy, generates SBOM, then pushes to DockerHub
- **Update manifest** — `sed` replaces the image tag in `k8-manifests/qa/app-deployment.yaml` and commits back to the `qa` branch
- **Deploy** — uses GitHub OIDC to assume `GitHubActionsEKSDeployRoleQA`, generates kubeconfig, applies manifests

### How the PROD pipeline works (`prod-cd.yaml`)

Triggered on push to `main` when `k8-manifests/prod/app-deployment.yaml` changes.

**No new build happens.** The pipeline:
1. Pulls `jayyp2op/3-tier-user-platform-project:latest` (the image that passed QA)
2. Retags it as `jayyp2op/3-tier-user-platform-project:prod-{sha}`
3. Pushes the prod-tagged image
4. Updates `k8-manifests/prod/app-deployment.yaml` with the new tag
5. Deploys to the `prod` namespace using `GitHubActionsEKSDeployRolePROD`

### GitHub Secrets and Variables needed

Go to your repo → **Settings → Secrets and variables → Actions**:

**Secrets:**
| Name | Value |
|------|-------|
| `AWS_ROLE_TO_ASSUME` | `arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRoleQA` |
| `AWS_ROLE_TO_ASSUME1` | `arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRolePROD` |
| `DOCKERHUB_TOKEN` | Your DockerHub access token |

**Variables:**
| Name | Value |
|------|-------|
| `DOCKERHUB_USERNAME` | Your DockerHub username (e.g. `jayyp2op`) |

---

## Phase 13 — Observability Stack (Prometheus + Loki + Grafana)

All components are deployed via a single PowerShell script.

### Step 13.1 — Edit deploy.ps1 before running

Open `Monitoring/deploy.ps1` and change these variables at the top:

```powershell
$S3_BUCKET              = "your-unique-bucket-name"   # Must be globally unique
$AWS_REGION             = "ap-south-1"
$CLUSTER_NAME           = "my-cluster"
$MONITORING_NS          = "monitoring"
$QA_NS                  = "qa"
$GRAFANA_ADMIN_PASSWORD = "your-secure-password"
$NODE_ROLE_NAME         = "eksctl-my-cluster-nodegroup-...-NodeInstanceRole-..."
```

To find your node role name:
```powershell
aws iam list-roles `
  --query "Roles[?contains(RoleName,'NodeInstanceRole')].RoleName" `
  --output table
```

### Step 13.2 — Run the deployment script
```powershell
cd Monitoring
.\deploy.ps1
```

The script automatically:
- Creates `monitoring` namespace
- Creates Grafana admin secret
- Creates S3 bucket for Loki log storage
- Creates and attaches `LokiS3Policy` IAM policy to your node role
- Installs Prometheus (kube-prometheus-stack)
- Installs Loki (SingleBinary mode with S3 backend)
- Installs Grafana Alloy (DaemonSet for log and metric collection)
- Installs Grafana with pre-built dashboards and ALB ingress

### Step 13.3 — Verify all pods are running
```powershell
kubectl get pods -n monitoring
# Expected: prometheus, alertmanager, loki, alloy, grafana all Running
```

### Step 13.4 — Get Grafana URL
```powershell
kubectl get ingress grafana -n monitoring
# Copy the ADDRESS — wait 2-3 minutes for ALB to provision
```

### What each component does in detail

**Prometheus (`01-prometheus-values.yaml`):**
- Watches `monitoring` and `qa` namespaces for ServiceMonitors and PodMonitors
- Stores 15 days of metrics on a 20Gi EBS volume
- `enableRemoteWriteReceiver: true` allows Grafana Alloy to push metrics to it
- AlertManager is enabled for alert routing

**Loki (`02-loki-values.yaml`):**
- Runs in SingleBinary mode (one pod) — suitable for non-HA environments
- Stores all logs in S3 (`qa-demo-s3-777`) with 30-day retention
- Uses TSDB schema v13 for efficient storage

**Grafana Alloy (`03-alloy-values.yaml`):**
- Runs as a DaemonSet (one pod per node)
- Discovers all pods in the `qa` namespace
- Collects logs using `loki.source.kubernetes` (Kubernetes API, not file mounts — works across nodes)
- Scrapes metrics from pods annotated with `prometheus.io/scrape: "true"`
- Ships logs to Loki gateway, metrics to Prometheus remote-write endpoint

**Grafana (`04-grafana-values.yaml`):**
- Pre-configured datasources for Prometheus and Loki
- Pre-installed dashboards: Kubernetes cluster overview, node exporter, pod metrics, Loki logs, namespaces, AlertManager
- Plugins: `grafana-lokiexplore-app`, `grafana-exploretraces-app`
- Exposed via ALB ingress (HTTP only)

---

## Verification Checklist

Run through this checklist after completing all phases:

```powershell
# Cluster health
kubectl get nodes
kubectl get pods -A

# QA namespace
kubectl get pods -n qa
kubectl get svc -n qa
kubectl get ingress -n qa
kubectl get pvc -n qa
kubectl get secret mysql-secret -n qa
kubectl get secretstore -n qa
kubectl get externalsecret -n qa

# PROD namespace
kubectl get pods -n prod
kubectl get svc -n prod
kubectl get ingress -n prod
kubectl get pvc -n prod
kubectl get secret mysql-secret -n prod

# Infrastructure
kubectl get pods -n kube-system | findstr aws-load-balancer
kubectl get pods -n kube-system | findstr ebs
kubectl get pods -n external-secrets

# Monitoring
kubectl get pods -n monitoring
kubectl get ingress -n monitoring

# IAM
aws iam list-attached-role-policies --role-name GitHubActionsEKSDeployRoleQA
aws iam list-attached-role-policies --role-name GitHubActionsEKSDeployRolePROD
aws eks list-associated-access-policies --cluster-name my-cluster --region ap-south-1 --principal-arn arn:aws:iam::796197769514:role/GitHubActionsEKSDeployRoleQA
```

---

## Common Errors & Fixes

**`NoSuchEntity` when attaching policy**
- Wrong account ID in the ARN. Always verify with `aws sts get-caller-identity` and use that account ID everywhere.

**`Could not assume role with OIDC: Not authorized`**
- The `sub` condition in the trust policy is wrong. It must be `repo:owner/repo-name:*` not `repo:owner/repo-name/tree/branch`.
- Fix: Update the trust policy with `aws iam update-assume-role-policy`.

**`MalformedPolicyDocument` when updating trust policy**
- Encoding issue from PowerShell here-strings. Use the single-line `$policy = '...'` approach with `Out-File -Encoding ascii -NoNewline`.

**MySQL pod stuck in `Pending`**
- EBS CSI driver not installed or `ebs-sc` StorageClass not created. Run Phase 5 again.
- Check: `kubectl describe pvc -n qa` for events.

**ExternalSecret `READY: False`**
- `eso-qa-sa` service account doesn't have the right IRSA annotation, or the IAM policy resource ARN has the wrong account ID.
- Check: `kubectl describe externalsecret mysql-external-secret -n qa`

**ALB not provisioning (Ingress ADDRESS empty)**
- Public subnets not tagged, or ALB controller not running.
- Check: `kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller`

**`ResourceNotFoundException` when associating EKS access policy**
- Access entry doesn't exist yet. Run `aws eks create-access-entry` first (Phase 6.6).

**Loki not receiving logs**
- Alloy is dropping logs because of JSON parse stages. Do NOT add `stage.json` or `drop_malformed=true` in `03-alloy-values.yaml` — MySQL and Node.js logs are plain text, not JSON.

**GitHub Actions deploy fails with `Unauthorized`**
- The EKS access entry exists but the access policy was not associated. Run the `associate-access-policy` command (Phase 6.7 / 7.5).