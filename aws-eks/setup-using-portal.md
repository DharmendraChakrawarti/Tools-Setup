# 🖥️ EKS Setup Using AWS Console (Portal)

A step-by-step guide to create an EKS cluster entirely through the AWS Management Console — no CLI needed.

---

## Table of Contents

1. [Create IAM Roles](#step-1--create-iam-roles)
2. [Create VPC & Networking](#step-2--create-vpc--networking)
3. [Create EKS Cluster](#step-3--create-eks-cluster)
4. [Create Node Group](#step-4--create-node-group)
5. [Connect to the Cluster](#step-5--connect-to-the-cluster)
6. [Deploy a Sample App](#step-6--deploy-a-sample-app)
7. [Install Add-ons](#step-7--install-add-ons-from-console)
8. [Monitor & Logs](#step-8--monitoring--logs)
9. [Cleanup](#step-9--cleanup)

---

## Step 1 — Create IAM Roles

You need **two roles** before creating the cluster.

### A) EKS Cluster Role

1. Go to **IAM Console** → https://console.aws.amazon.com/iam
2. Click **Roles** → **Create role**
3. Select **AWS service** as trusted entity
4. Under **Use case**, choose **EKS** → **EKS - Cluster**
5. Click **Next**
6. The policy `AmazonEKSClusterPolicy` is auto-attached — verify it's listed
7. Click **Next**
8. Role name: `EKSClusterRole`
9. Click **Create role**

### B) Node Group Role

1. Click **Roles** → **Create role**
2. Select **AWS service** → Use case: **EC2**
3. Click **Next**
4. Search and attach these **3 policies**:
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonEKS_CNI_Policy`
   - `AmazonEC2ContainerRegistryReadOnly`
5. Click **Next**
6. Role name: `EKSNodeRole`
7. Click **Create role**

> ✅ You should now have two roles: `EKSClusterRole` and `EKSNodeRole`

---

## Step 2 — Create VPC & Networking

### Option A: Use CloudFormation Template (Recommended)

1. Go to **CloudFormation Console** → https://console.aws.amazon.com/cloudformation
2. Click **Create stack** → **With new resources (standard)**
3. Select **Amazon S3 URL** and paste:
   ```
   https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
   ```
4. Click **Next**
5. Stack name: `eks-vpc`
6. Leave default parameters or customize:
   - VPC CIDR: `10.0.0.0/16`
   - Public Subnet 1: `10.0.1.0/24`
   - Public Subnet 2: `10.0.2.0/24`
   - Private Subnet 1: `10.0.10.0/24`
   - Private Subnet 2: `10.0.20.0/24`
7. Click **Next** → **Next** → **Submit**
8. Wait for status: `CREATE_COMPLETE` (~3 minutes)
9. Go to **Outputs** tab — note down:
   - **VpcId**
   - **SubnetIds**
   - **SecurityGroups**

### Option B: Create VPC Manually

1. Go to **VPC Console** → https://console.aws.amazon.com/vpc
2. Click **Create VPC**
3. Select **VPC and more**
4. Configure:

   | Setting | Value |
   |---|---|
   | Name tag | `eks-vpc` |
   | IPv4 CIDR | `10.0.0.0/16` |
   | Number of AZs | `2` (minimum) or `3` (recommended) |
   | Public subnets | `2` or `3` |
   | Private subnets | `2` or `3` |
   | NAT gateways | `In 1 AZ` (dev) or `1 per AZ` (prod) |
   | VPC endpoints | `S3 Gateway` |

5. Click **Create VPC**
6. Wait for all resources to be created

### Tag the Subnets (Important!)

1. Go to **VPC** → **Subnets**
2. Select each **public subnet** → **Tags** → **Manage tags** → Add:
   - Key: `kubernetes.io/role/elb` Value: `1`
3. Select each **private subnet** → **Tags** → **Manage tags** → Add:
   - Key: `kubernetes.io/role/internal-elb` Value: `1`

---

## Step 3 — Create EKS Cluster

1. Go to **EKS Console** → https://console.aws.amazon.com/eks
2. Click **Add cluster** → **Create**

### 3.1 Configure Cluster

| Setting | Value |
|---|---|
| Name | `my-cluster` |
| Kubernetes version | `1.30` (latest stable) |
| Cluster service role | `EKSClusterRole` (created in Step 1) |

Click **Next**

### 3.2 Specify Networking

| Setting | Value |
|---|---|
| VPC | Select `eks-vpc` (created in Step 2) |
| Subnets | Select all public + private subnets |
| Security groups | Leave default (EKS creates one) |
| Cluster endpoint access | **Public and private** |

Click **Next**

### 3.3 Configure Observability

| Setting | Value |
|---|---|
| Control plane logging | Enable **all 5 types**: API server, Audit, Authenticator, Controller manager, Scheduler |
| CloudWatch Container Insights | Toggle **ON** |

Click **Next**

### 3.4 Select Add-ons

Keep the default add-ons selected:
- ✅ Amazon VPC CNI
- ✅ CoreDNS
- ✅ kube-proxy

Optionally add:
- ✅ Amazon EBS CSI Driver (for persistent volumes)

Click **Next**

### 3.5 Configure Add-on Versions

Leave defaults (latest versions) → Click **Next**

### 3.6 Review and Create

Review all settings → Click **Create**

> ⏳ Cluster creation takes **10–15 minutes**. Status will change from `Creating` → `Active`.

---

## Step 4 — Create Node Group

Once the cluster status is **Active**:

1. Click on your cluster name → **Compute** tab
2. Under **Node groups**, click **Add node group**

### 4.1 Configure Node Group

| Setting | Value |
|---|---|
| Name | `workers` |
| Node IAM role | `EKSNodeRole` (created in Step 1) |

Click **Next**

### 4.2 Set Compute and Scaling

| Setting | Value |
|---|---|
| AMI type | `Amazon Linux 2023 (AL2023_x86_64_STANDARD)` |
| Capacity type | `On-Demand` |
| Instance type | `t3.medium` (dev) or `m6i.xlarge` (prod) |
| Disk size | `20 GB` (dev) or `100 GB` (prod) |

**Scaling configuration:**

| Setting | Value |
|---|---|
| Minimum size | `1` |
| Maximum size | `4` |
| Desired size | `2` |

Click **Next**

### 4.3 Specify Networking

| Setting | Value |
|---|---|
| Subnets | Select **private subnets only** |
| SSH access | Optional — select a key pair if needed |

Click **Next**

### 4.4 Review and Create

Review → Click **Create**

> ⏳ Node group creation takes **3–5 minutes**. Wait until status is `Active`.

---

## Step 5 — Connect to the Cluster

You need `kubectl` and `AWS CLI` installed locally to connect. (See [simple-setup.md](./simple-setup.md#step-1--install-tools) for install steps)

```bash
# Configure kubeconfig
aws eks update-kubeconfig \
  --name my-cluster \
  --region ap-south-1

# Verify connection
kubectl get nodes
```

Expected output:
```
NAME                             STATUS   ROLES    AGE   VERSION
ip-10-0-10-xx.ec2.internal      Ready    <none>   3m    v1.30.x
ip-10-0-20-xx.ec2.internal      Ready    <none>   3m    v1.30.x
```

### Grant Access to Other Users (Optional)

1. Go to **EKS Console** → Your cluster → **Access** tab
2. Click **Create access entry**
3. Select the **IAM principal** (user or role)
4. Choose access policy:
   - `AmazonEKSClusterAdminPolicy` — full admin
   - `AmazonEKSAdminViewPolicy` — read-only
5. Click **Create**

---

## Step 6 — Deploy a Sample App

```bash
# Create a namespace
kubectl create namespace demo

# Deploy nginx
kubectl create deployment nginx --image=nginx:1.27 -n demo --replicas=2

# Expose via LoadBalancer
kubectl expose deployment nginx -n demo --type=LoadBalancer --port=80

# Get the external URL
kubectl get svc nginx -n demo -w
```

Copy the `EXTERNAL-IP` and open it in your browser — you'll see the Nginx welcome page.

---

## Step 7 — Install Add-ons from Console

1. Go to **EKS Console** → Your cluster → **Add-ons** tab
2. Click **Get more add-ons**

### Recommended Add-ons

| Add-on | Purpose | Action |
|---|---|---|
| **Amazon EBS CSI Driver** | Persistent block storage | Select → Next → Create |
| **Amazon EFS CSI Driver** | Shared file storage | Select → Next → Create |
| **CoreDNS** | DNS resolution (pre-installed) | Verify active |
| **kube-proxy** | Network proxy (pre-installed) | Verify active |
| **VPC CNI** | Pod networking (pre-installed) | Verify active |
| **CloudWatch Observability** | Logs & metrics | Select → Next → Create |

### For Each Add-on:

1. Select the add-on → Click **Next**
2. Choose version: **Latest**
3. IAM role: Select an existing IRSA role or let EKS use the node role
4. Click **Create**

### Create OIDC Provider (Required for IRSA)

This step must be done via CLI:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region ap-south-1 \
  --approve
```

Or via Console:
1. Go to **EKS** → Your cluster → **Overview** tab
2. Copy the **OpenID Connect provider URL**
3. Go to **IAM** → **Identity providers** → **Add provider**
4. Provider type: **OpenID Connect**
5. Provider URL: paste the URL copied above
6. Audience: `sts.amazonaws.com`
7. Click **Add provider**

---

## Step 8 — Monitoring & Logs

### CloudWatch Container Insights

1. Go to **CloudWatch Console** → https://console.aws.amazon.com/cloudwatch
2. Left sidebar → **Insights** → **Container Insights**
3. Select your cluster from the dropdown
4. View:
   - **Cluster metrics** — CPU, memory, network
   - **Node metrics** — per-node resource usage
   - **Pod metrics** — per-pod performance
   - **Service metrics** — service-level overview

### Control Plane Logs

1. Go to **CloudWatch** → **Log groups**
2. Find: `/aws/eks/my-cluster/cluster`
3. Log streams available:
   - `kube-apiserver-*`
   - `kube-controller-manager-*`
   - `kube-scheduler-*`
   - `authenticator-*`
   - `cloud-controller-manager-*`

### Cost Monitoring

1. Go to **AWS Cost Explorer** → https://console.aws.amazon.com/cost-management
2. Filter by tag: `eks:cluster-name = my-cluster`
3. View breakdown by:
   - EC2 instances (nodes)
   - NAT Gateway
   - EBS volumes
   - Load Balancers

---

## Step 9 — Cleanup

### Delete Node Group

1. Go to **EKS Console** → Your cluster → **Compute** tab
2. Select node group → **Delete**
3. Type the node group name to confirm
4. Wait for deletion (~3–5 minutes)

### Delete the Cluster

1. Go to **EKS Console** → Select your cluster
2. Click **Delete cluster**
3. Type the cluster name to confirm
4. Wait for deletion (~5–10 minutes)

### Delete the VPC

1. Go to **CloudFormation Console** (if you used the template)
2. Select `eks-vpc` stack → **Delete**
3. Or go to **VPC Console** → Select VPC → **Actions** → **Delete VPC**

### Delete IAM Roles

1. Go to **IAM** → **Roles**
2. Search and delete:
   - `EKSClusterRole`
   - `EKSNodeRole`

> ⚠️ **Also check for leftover resources:**
> - Elastic IPs (**VPC** → **Elastic IPs**)
> - Load Balancers (**EC2** → **Load Balancers**)
> - EBS Volumes (**EC2** → **Volumes**)
> - NAT Gateways (**VPC** → **NAT Gateways**)
> - CloudWatch Log Groups (**CloudWatch** → **Log groups** → delete `/aws/eks/my-cluster/*`)

---

## Console vs CLI — Quick Comparison

| Task | Console | CLI/eksctl |
|---|---|---|
| Create cluster | ~15 clicks, 15 min | 1 command, 15 min |
| Create node group | ~10 clicks, 5 min | 1 command, 5 min |
| Install add-ons | 3 clicks per add-on | 1 command per add-on |
| View logs | CloudWatch UI | `kubectl logs` |
| Scaling | Edit node group settings | `kubectl scale` / HPA |
| Upgrade version | Click upgrade button | `eksctl upgrade cluster` |
| Best for | Learning, one-time setup | Automation, repeatability |

---

## Common Console Locations

| What | Where |
|---|---|
| EKS Dashboard | `EKS` → Clusters |
| Cluster details | `EKS` → Cluster name |
| Node groups | `EKS` → Cluster → Compute tab |
| Add-ons | `EKS` → Cluster → Add-ons tab |
| Access management | `EKS` → Cluster → Access tab |
| Networking | `EKS` → Cluster → Networking tab |
| Logs | `CloudWatch` → Log groups → `/aws/eks/` |
| Metrics | `CloudWatch` → Container Insights |
| IAM Roles | `IAM` → Roles |
| VPC & Subnets | `VPC` → Your VPCs / Subnets |
| EC2 Nodes | `EC2` → Instances (filter by cluster tag) |
| Load Balancers | `EC2` → Load Balancers |
| ECR (Images) | `ECR` → Repositories |

---

## What's Next?

- **CLI setup** → [simple-setup.md](./simple-setup.md)
- **Full production guide** → [EKS-Setup-Guide.md](./EKS-Setup-Guide.md)
- **Terraform (IaC)** → [EKS-Setup-Guide.md#20-terraform-iac-reference](./EKS-Setup-Guide.md#20-terraform-iac-reference)

---

*Last updated: November 2026*
