# 🚀 Production-Level Amazon EKS Setup Guide

> **IMPORTANT:** This guide covers end-to-end EKS cluster provisioning on AWS with production-grade best practices including networking, security, observability, autoscaling, and CI/CD readiness.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [AWS Account & IAM Setup](#2-aws-account--iam-setup)
3. [Networking — VPC & Subnets](#3-networking--vpc--subnets)
4. [EKS Cluster Creation](#4-eks-cluster-creation)
5. [Node Groups (Managed & Self-Managed)](#5-node-groups)
6. [kubectl & kubeconfig Setup](#6-kubectl--kubeconfig-setup)
7. [IAM Roles for Service Accounts (IRSA)](#7-iam-roles-for-service-accounts-irsa)
8. [Storage — EBS & EFS CSI Drivers](#8-storage--ebs--efs-csi-drivers)
9. [Ingress — AWS Load Balancer Controller](#9-ingress--aws-load-balancer-controller)
10. [DNS — ExternalDNS with Route 53](#10-dns--externaldns-with-route-53)
11. [TLS / Certificate Management](#11-tls--certificate-management)
12. [Autoscaling — Cluster Autoscaler & Karpenter](#12-autoscaling--cluster-autoscaler--karpenter)
13. [Observability — Logging, Metrics & Tracing](#13-observability--logging-metrics--tracing)
14. [Security Hardening](#14-security-hardening)
15. [Secrets Management](#15-secrets-management)
16. [Backup & Disaster Recovery](#16-backup--disaster-recovery)
17. [CI/CD Integration](#17-cicd-integration)
18. [Cost Optimization](#18-cost-optimization)
19. [Production Checklist](#19-production-checklist)
20. [Terraform (IaC) Reference](#20-terraform-iac-reference)

---

## 1. Prerequisites

### Tools to Install

```bash
# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws --version

# kubectl (match your EKS K8s version)
curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client

# eksctl — official EKS CLI
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Helm 3
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# Terraform (optional, recommended for IaC)
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

### AWS CLI Configuration

```bash
aws configure
# AWS Access Key ID:     <YOUR_ACCESS_KEY>
# AWS Secret Access Key: <YOUR_SECRET_KEY>
# Default region name:   ap-south-1
# Default output format: json

# Verify identity
aws sts get-caller-identity
```

---

## 2. AWS Account & IAM Setup

### 2.1 Create a Dedicated EKS Admin IAM Role

```bash
# Create the trust policy
cat > eks-admin-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:root" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name EKSClusterAdmin \
  --assume-role-policy-document file://eks-admin-trust-policy.json

# Attach necessary policies
aws iam attach-role-policy --role-name EKSClusterAdmin \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam attach-role-policy --role-name EKSClusterAdmin \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSServicePolicy
```

### 2.2 EKS Cluster Service Role

```bash
cat > eks-cluster-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "eks.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name EKSClusterRole \
  --assume-role-policy-document file://eks-cluster-trust.json

aws iam attach-role-policy --role-name EKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam attach-role-policy --role-name EKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController
```

### 2.3 Node Group IAM Role

```bash
cat > node-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name EKSNodeRole \
  --assume-role-policy-document file://node-trust.json

# Attach required node policies
for policy in \
  arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy \
  arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
  arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly \
  arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore; do
  aws iam attach-role-policy --role-name EKSNodeRole --policy-arn "$policy"
done
```

---

## 3. Networking — VPC & Subnets

### Production VPC Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │              VPC  10.0.0.0/16               │
                        │                                             │
  ┌─────────────────────┼─────────────────────────────────────────────┼──────────────────┐
  │  Public Subnets     │                                             │                  │
  │  ┌──────────────┐   │  ┌──────────────┐   ┌──────────────┐       │                  │
  │  │  AZ-a        │   │  │  AZ-b        │   │  AZ-c        │       │                  │
  │  │  10.0.1.0/24 │   │  │  10.0.2.0/24 │   │  10.0.3.0/24 │       │                  │
  │  └──────┬───────┘   │  └──────┬───────┘   └──────┬───────┘       │                  │
  │         │           │         │                   │               │                  │
  │         └───────────┼─────────┴───────────────────┘               │                  │
  │                     │              │                               │                  │
  │                     │      ┌───────▼───────┐                      │                  │
  │                     │      │ Internet GW   │                      │                  │
  │                     │      └───────┬───────┘                      │                  │
  │                     │              │                               │                  │
  │                     │      ┌───────▼───────┐                      │                  │
  │                     │      │  NAT Gateway  │                      │                  │
  │                     │      └───────┬───────┘                      │                  │
  │                     │              │                               │                  │
  │  Private Subnets    │              │                               │                  │
  │  ┌──────────────┐   │  ┌──────────────┐   ┌──────────────┐       │                  │
  │  │  AZ-a        │   │  │  AZ-b        │   │  AZ-c        │       │                  │
  │  │ 10.0.10.0/24 │   │  │ 10.0.20.0/24 │   │ 10.0.30.0/24 │       │                  │
  │  └──────────────┘   │  └──────────────┘   └──────────────┘       │                  │
  └─────────────────────┼─────────────────────────────────────────────┼──────────────────┘
                        └─────────────────────────────────────────────┘
```

### 3.1 Create VPC with eksctl (Quickstart)

```bash
eksctl create cluster \
  --name production-cluster \
  --region ap-south-1 \
  --version 1.30 \
  --without-nodegroup \
  --vpc-cidr 10.0.0.0/16 \
  --vpc-nat-mode HighlyAvailable
```

### 3.2 Create VPC Manually (Recommended for Production)

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=eks-prod-vpc}]' \
  --query 'Vpc.VpcId' --output text)

# Enable DNS
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=eks-prod-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# --- Public Subnets (3 AZs) ---
PUB_SUB_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-pub-a},{Key=kubernetes.io/role/elb,Value=1}]' \
  --query 'Subnet.SubnetId' --output text)

PUB_SUB_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-pub-b},{Key=kubernetes.io/role/elb,Value=1}]' \
  --query 'Subnet.SubnetId' --output text)

PUB_SUB_C=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 \
  --availability-zone ap-south-1c \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-pub-c},{Key=kubernetes.io/role/elb,Value=1}]' \
  --query 'Subnet.SubnetId' --output text)

# Enable auto-assign public IP on public subnets
for sub in $PUB_SUB_A $PUB_SUB_B $PUB_SUB_C; do
  aws ec2 modify-subnet-attribute --subnet-id $sub --map-public-ip-on-launch
done

# --- Private Subnets (3 AZs) ---
PRIV_SUB_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.10.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-priv-a},{Key=kubernetes.io/role/internal-elb,Value=1}]' \
  --query 'Subnet.SubnetId' --output text)

PRIV_SUB_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.20.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-priv-b},{Key=kubernetes.io/role/internal-elb,Value=1}]' \
  --query 'Subnet.SubnetId' --output text)

PRIV_SUB_C=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.30.0/24 \
  --availability-zone ap-south-1c \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-priv-c},{Key=kubernetes.io/role/internal-elb,Value=1}]' \
  --query 'Subnet.SubnetId' --output text)

# --- NAT Gateway (HA — one per AZ) ---
EIP_A=$(aws ec2 allocate-address --query 'AllocationId' --output text)
NAT_A=$(aws ec2 create-nat-gateway --subnet-id $PUB_SUB_A --allocation-id $EIP_A \
  --query 'NatGateway.NatGatewayId' --output text)
# Repeat for AZ-b and AZ-c for true HA (or use single NAT for cost saving)

# --- Route Tables ---
# Public route table
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
for sub in $PUB_SUB_A $PUB_SUB_B $PUB_SUB_C; do
  aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $sub
done

# Private route table (through NAT)
PRIV_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PRIV_RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_A
for sub in $PRIV_SUB_A $PRIV_SUB_B $PRIV_SUB_C; do
  aws ec2 associate-route-table --route-table-id $PRIV_RT --subnet-id $sub
done
```

> **TIP: Subnet tagging is critical for EKS:**
> - Public subnets: `kubernetes.io/role/elb = 1`
> - Private subnets: `kubernetes.io/role/internal-elb = 1`
> - All subnets: `kubernetes.io/cluster/<cluster-name> = shared`

---

## 4. EKS Cluster Creation

### 4.1 Using eksctl (Recommended Quick Method)

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production-cluster
  region: ap-south-1
  version: "1.30"
  tags:
    Environment: production
    Team: platform

kubernetesNetworkConfig:
  ipFamily: IPv4
  serviceIPv4CIDR: "172.20.0.0/16"

vpc:
  cidr: "10.0.0.0/16"
  clusterEndpoints:
    publicAccess: true
    privateAccess: true
  nat:
    gateway: HighlyAvailable    # One NAT GW per AZ

iam:
  withOIDC: true                # Required for IRSA

cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler
    logRetentionInDays: 90

addons:
  - name: vpc-cni
    version: latest
    configurationValues: '{"enableNetworkPolicy": "true"}'
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    wellKnownPolicies:
      ebsCSIController: true
  - name: aws-efs-csi-driver
    version: latest

managedNodeGroups:
  - name: system-ng
    instanceType: m6i.xlarge
    minSize: 2
    maxSize: 5
    desiredCapacity: 3
    volumeSize: 100
    volumeType: gp3
    privateNetworking: true
    labels:
      role: system
    tags:
      nodegroup-role: system
    iam:
      withAddonPolicies:
        imageBuilder: true
        autoScaler: true
        ebs: true
        efs: true
        cloudWatch: true

  - name: app-ng
    instanceType: m6i.2xlarge
    minSize: 3
    maxSize: 20
    desiredCapacity: 5
    volumeSize: 100
    volumeType: gp3
    privateNetworking: true
    labels:
      role: application
    taints:
      - key: dedicated
        value: application
        effect: NoSchedule
    tags:
      nodegroup-role: application
    iam:
      withAddonPolicies:
        imageBuilder: true
        autoScaler: true
        ebs: true
        efs: true
        cloudWatch: true
```

```bash
eksctl create cluster -f cluster-config.yaml
```

### 4.2 Using AWS CLI (Manual)

```bash
# Create the cluster
aws eks create-cluster \
  --name production-cluster \
  --region ap-south-1 \
  --kubernetes-version 1.30 \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/EKSClusterRole \
  --resources-vpc-config \
    subnetIds=$PRIV_SUB_A,$PRIV_SUB_B,$PRIV_SUB_C,$PUB_SUB_A,$PUB_SUB_B,$PUB_SUB_C,\
    securityGroupIds=$CLUSTER_SG,\
    endpointPublicAccess=true,\
    endpointPrivateAccess=true \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Wait for cluster to be ACTIVE
aws eks wait cluster-active --name production-cluster

# Enable OIDC provider (required for IRSA)
eksctl utils associate-iam-oidc-provider \
  --cluster production-cluster \
  --approve
```

---

## 5. Node Groups

### 5.1 Managed Node Group

```bash
aws eks create-nodegroup \
  --cluster-name production-cluster \
  --nodegroup-name app-nodes \
  --node-role arn:aws:iam::<ACCOUNT_ID>:role/EKSNodeRole \
  --subnets $PRIV_SUB_A $PRIV_SUB_B $PRIV_SUB_C \
  --instance-types m6i.2xlarge \
  --scaling-config minSize=3,maxSize=20,desiredSize=5 \
  --disk-size 100 \
  --capacity-type ON_DEMAND \
  --labels role=application \
  --tags Environment=production
```

### 5.2 Spot Instance Node Group (for non-critical workloads)

```bash
aws eks create-nodegroup \
  --cluster-name production-cluster \
  --nodegroup-name spot-nodes \
  --node-role arn:aws:iam::<ACCOUNT_ID>:role/EKSNodeRole \
  --subnets $PRIV_SUB_A $PRIV_SUB_B $PRIV_SUB_C \
  --instance-types m6i.xlarge m6i.2xlarge m5.xlarge m5.2xlarge \
  --scaling-config minSize=0,maxSize=50,desiredSize=3 \
  --capacity-type SPOT \
  --labels role=batch,lifecycle=spot \
  --taints "key=lifecycle,value=spot,effect=NoSchedule"
```

### 5.3 Fargate Profile (Serverless)

```bash
aws eks create-fargate-profile \
  --cluster-name production-cluster \
  --fargate-profile-name serverless-profile \
  --pod-execution-role-arn arn:aws:iam::<ACCOUNT_ID>:role/EKSFargatePodRole \
  --subnets $PRIV_SUB_A $PRIV_SUB_B $PRIV_SUB_C \
  --selectors namespace=serverless
```

---

## 6. kubectl & kubeconfig Setup

```bash
# Update kubeconfig
aws eks update-kubeconfig \
  --name production-cluster \
  --region ap-south-1 \
  --alias production

# Verify
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns

# (Optional) Enable kubectl aliases
cat >> ~/.bashrc <<'EOF'
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
EOF
source ~/.bashrc
```

### Access Management with aws-auth ConfigMap

```bash
# Grant other IAM users/roles access
kubectl edit -n kube-system configmap/aws-auth
```

```yaml
# aws-auth ConfigMap example
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::<ACCOUNT_ID>:role/EKSNodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::<ACCOUNT_ID>:role/DevOpsEngineer
      username: devops-engineer
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::<ACCOUNT_ID>:user/admin
      username: admin
      groups:
        - system:masters
```

> **NOTE:** Starting with EKS 1.24+, AWS recommends **EKS Access Entries** over `aws-auth` ConfigMap for access management. See [EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html).

---

## 7. IAM Roles for Service Accounts (IRSA)

IRSA allows Kubernetes pods to assume IAM roles securely without embedding credentials.

```bash
# 1. Create an IAM policy for your app
cat > s3-read-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-app-bucket", "arn:aws:s3:::my-app-bucket/*"]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name EKS-S3ReadAccess \
  --policy-document file://s3-read-policy.json

# 2. Create the IRSA role
eksctl create iamserviceaccount \
  --cluster production-cluster \
  --namespace my-app \
  --name my-app-sa \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/EKS-S3ReadAccess \
  --approve \
  --override-existing-serviceaccounts

# 3. Use in your deployment
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app-sa   # <-- Uses IRSA
      containers:
        - name: app
          image: <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
```

---

## 8. Storage — EBS & EFS CSI Drivers

### 8.1 EBS CSI Driver (Block Storage)

```bash
# Install the EBS CSI add-on (if not done via eksctl)
eksctl create addon \
  --cluster production-cluster \
  --name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole
```

```yaml
# StorageClass for gp3 (recommended)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Retain          # Production: Retain to prevent data loss
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### 8.2 EFS CSI Driver (Shared File System)

```bash
# Install the EFS CSI add-on
eksctl create addon \
  --cluster production-cluster \
  --name aws-efs-csi-driver

# Create EFS filesystem
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=eks-efs \
  --query 'FileSystemId' --output text)

# Create mount targets in each private subnet
for sub in $PRIV_SUB_A $PRIV_SUB_B $PRIV_SUB_C; do
  aws efs create-mount-target \
    --file-system-id $EFS_ID \
    --subnet-id $sub \
    --security-groups $EFS_SG
done
```

```yaml
# EFS StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxxxxxx
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
```

---

## 9. Ingress — AWS Load Balancer Controller

```bash
# 1. Create IAM policy
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.1/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 2. Create IRSA
eksctl create iamserviceaccount \
  --cluster production-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# 3. Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=production-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=$VPC_ID
```

### Sample Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: my-app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:<ACCOUNT_ID>:certificate/<CERT_ID>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:...:webacl/prod-waf/...
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-svc
                port:
                  number: 80
```

---

## 10. DNS — ExternalDNS with Route 53

```bash
# Create IRSA for ExternalDNS
eksctl create iamserviceaccount \
  --cluster production-cluster \
  --namespace kube-system \
  --name external-dns \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess \
  --approve

# Install ExternalDNS
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm install external-dns external-dns/external-dns \
  -n kube-system \
  --set provider=aws \
  --set aws.zoneType=public \
  --set domainFilters[0]=example.com \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-dns \
  --set policy=sync \
  --set txtOwnerId=production-cluster
```

---

## 11. TLS / Certificate Management

### Option A: AWS ACM (for ALB Ingress — Recommended)

```bash
# Request a certificate
aws acm request-certificate \
  --domain-name "*.example.com" \
  --validation-method DNS \
  --subject-alternative-names "example.com"

# Validate via Route 53 (auto)
aws acm describe-certificate --certificate-arn <CERT_ARN>
```

### Option B: cert-manager (for in-cluster TLS)

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true
```

```yaml
# ClusterIssuer with Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: devops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - dns01:
          route53:
            region: ap-south-1
```

---

## 12. Autoscaling — Cluster Autoscaler & Karpenter

### 12.1 Cluster Autoscaler

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=production-cluster \
  --set awsRegion=ap-south-1 \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.expander=least-waste \
  --set extraArgs.scale-down-delay-after-add=5m \
  --set extraArgs.scale-down-unneeded-time=5m
```

### 12.2 Karpenter (Recommended for Production)

```bash
# Set environment variables
export KARPENTER_VERSION="1.0.0"
export CLUSTER_NAME="production-cluster"

# Install Karpenter
helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace kube-system \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --wait
```

```yaml
# Karpenter NodePool
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        managed-by: karpenter
    spec:
      requirements:
        - key: "karpenter.k8s.aws/instance-category"
          operator: In
          values: ["m", "c", "r"]
        - key: "karpenter.k8s.aws/instance-generation"
          operator: Gt
          values: ["5"]
        - key: "kubernetes.io/arch"
          operator: In
          values: ["amd64"]
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["on-demand", "spot"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: "1000"
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
    - alias: al2023@latest
  subnetSelectorTerms:
    - tags:
        kubernetes.io/role/internal-elb: "1"
  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/production-cluster: owned
  role: "KarpenterNodeRole-production-cluster"
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true
```

### 12.3 Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---

## 13. Observability — Logging, Metrics & Tracing

### Architecture Overview

```
  Application Pods
       │
       ├──── Fluent Bit (DaemonSet) ──→ CloudWatch Logs / OpenSearch
       │
       └──── OTel Collector ──→ Amazon Managed Prometheus ──→ Managed Grafana
                     │
                     └──→ AWS X-Ray (Distributed Tracing)
```

### 13.1 Amazon CloudWatch Container Insights

```bash
# Install CloudWatch Observability Add-on
aws eks create-addon \
  --cluster-name production-cluster \
  --addon-name amazon-cloudwatch-observability
```

### 13.2 Prometheus & Grafana Stack

```bash
# Amazon Managed Prometheus (AMP)
aws amp create-workspace --alias production-metrics

# Install Prometheus with remote_write to AMP
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set prometheus.prometheusSpec.remoteWrite[0].url="https://aps-workspaces.ap-south-1.amazonaws.com/workspaces/<WORKSPACE_ID>/api/v1/remote_write" \
  --set prometheus.prometheusSpec.remoteWrite[0].sigv4.region=ap-south-1 \
  --set prometheus.prometheusSpec.retention=2d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=gp3 \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.enabled=true \
  --set alertmanager.enabled=true
```

### 13.3 Fluent Bit (Logging)

```bash
helm repo add fluent https://fluent.github.io/helm-charts

helm install fluent-bit fluent/fluent-bit \
  -n logging --create-namespace \
  --set config.outputs="|
    [OUTPUT]
        Name              cloudwatch_logs
        Match             *
        region            ap-south-1
        log_group_name    /eks/production-cluster/application
        log_stream_prefix fluentbit-
        auto_create_group true
        log_retention_days 90"
```

---

## 14. Security Hardening

### 14.1 Pod Security Standards

```yaml
# Enforce restricted pod security on namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 14.2 Network Policies

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Allow traffic only from specific namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-system
      ports:
        - protocol: TCP
          port: 8080
```

### 14.3 Security Group for Pods (SGP)

```bash
# Enable SGP on the VPC CNI
kubectl set env daemonset aws-node \
  -n kube-system \
  ENABLE_POD_ENI=true
```

### 14.4 OPA Gatekeeper / Kyverno

```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno \
  -n kyverno --create-namespace \
  --set replicaCount=3
```

```yaml
# Kyverno policy: Disallow latest tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Using 'latest' tag is not allowed."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

### 14.5 Runtime Security — Falco

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  -n falco --create-namespace \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/services/..."
```

### 14.6 Image Scanning — ECR

```bash
# Enable scan-on-push for ECR repos
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true

# Enable enhanced scanning (Inspector)
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{"scanFrequency":"CONTINUOUS_SCAN","repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}]}]'
```

> **⚠️ CAUTION: Never run containers as root in production.** Always set `runAsNonRoot: true` and `readOnlyRootFilesystem: true` in your security context.

```yaml
# Production-grade security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000
  seccompProfile:
    type: RuntimeDefault
containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

---

## 15. Secrets Management

### 15.1 AWS Secrets Manager + CSI Driver

```bash
# Install Secrets Store CSI Driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  -n kube-system \
  --set syncSecret.enabled=true \
  --set enableSecretRotation=true

# Install AWS provider
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

```yaml
# SecretProviderClass
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  namespace: my-app
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/my-app/db-credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: db_username
          - path: password
            objectAlias: db_password
  secretObjects:
    - secretName: db-credentials
      type: Opaque
      data:
        - objectName: db_username
          key: username
        - objectName: db_password
          key: password
```

### 15.2 External Secrets Operator (Alternative)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

---

## 16. Backup & Disaster Recovery

### 16.1 Velero for Cluster Backup

```bash
# Install Velero with AWS provider
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket eks-backup-bucket \
  --backup-location-config region=ap-south-1 \
  --snapshot-location-config region=ap-south-1 \
  --secret-file ./credentials-velero

# Schedule daily backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h \
  --include-namespaces my-app,my-app-staging

# Manual backup
velero backup create pre-upgrade-backup \
  --include-namespaces my-app

# Restore
velero restore create --from-backup daily-backup-20261119020000
```

### 16.2 Multi-Region DR Strategy

```
  Primary Region (ap-south-1)              DR Region (us-west-2)
  ┌────────────────────────┐              ┌────────────────────────┐
  │  EKS Cluster           │              │  EKS Cluster (Standby) │
  │  RDS Primary           │──Async Rep──▶│  RDS Read Replica      │
  │  S3 Bucket             │──CRR──────▶  │  S3 Replica            │
  └────────────────────────┘              └────────────────────────┘
            ▲                                        ▲
            │            Route 53                    │
            └──── Failover Routing ──────────────────┘
```

---

## 17. CI/CD Integration

### 17.1 GitHub Actions Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build & Deploy to EKS

on:
  push:
    branches: [main]

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: my-app
  EKS_CLUSTER: production-cluster

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsEKSRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: Scan image
        run: |
          aws ecr start-image-scan \
            --repository-name $ECR_REPOSITORY \
            --image-id imageTag=${{ github.sha }}

      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name $EKS_CLUSTER --region $AWS_REGION
          kubectl set image deployment/my-app \
            app=${{ steps.ecr-login.outputs.registry }}/$ECR_REPOSITORY:${{ github.sha }} \
            -n my-app
          kubectl rollout status deployment/my-app -n my-app --timeout=300s
```

### 17.2 ArgoCD (GitOps)

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose via ALB
kubectl patch svc argocd-server -n argocd \
  -p '{"metadata":{"annotations":{"service.beta.kubernetes.io/aws-load-balancer-type":"external","service.beta.kubernetes.io/aws-load-balancer-scheme":"internet-facing"}},"spec":{"type":"LoadBalancer"}}'

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## 18. Cost Optimization

| Strategy | Implementation | Savings |
|---|---|---|
| **Spot Instances** | Karpenter with spot capacity-type | 60-90% on compute |
| **Right-sizing** | VPA + Goldilocks | 20-40% |
| **Graviton (ARM)** | Use `m7g`/`c7g` instances | ~20% over x86 |
| **Savings Plans** | Commit to 1-3 yr compute usage | 30-60% |
| **Scale-to-zero** | KEDA for event-driven workloads | Variable |
| **Pod Density** | Optimize resource requests | 10-30% |
| **NAT Gateway** | Use VPC endpoints for AWS services | \$0.045/GB saved |

### VPC Endpoints (Reduce NAT costs)

```bash
# Create essential VPC endpoints
for svc in com.amazonaws.ap-south-1.s3 \
            com.amazonaws.ap-south-1.ecr.api \
            com.amazonaws.ap-south-1.ecr.dkr \
            com.amazonaws.ap-south-1.ec2 \
            com.amazonaws.ap-south-1.sts \
            com.amazonaws.ap-south-1.logs \
            com.amazonaws.ap-south-1.elasticloadbalancing; do
  aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --service-name $svc \
    --vpc-endpoint-type Interface \
    --subnet-ids $PRIV_SUB_A $PRIV_SUB_B $PRIV_SUB_C \
    --security-group-ids $VPCE_SG
done

# S3 Gateway Endpoint (free!)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids $PRIV_RT
```

### Vertical Pod Autoscaler (VPA) + Goldilocks

```bash
# Install VPA
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install vpa fairwinds-stable/vpa -n vpa --create-namespace

# Install Goldilocks (dashboard for right-sizing)
helm install goldilocks fairwinds-stable/goldilocks -n goldilocks --create-namespace

# Enable Goldilocks on a namespace
kubectl label ns my-app goldilocks.fairwinds.com/enabled=true
```

---

## 19. Production Checklist

### Cluster

- [ ] Kubernetes version >= 1.29 (latest stable)
- [ ] Private API endpoint enabled
- [ ] Control plane logging enabled (all 5 log types)
- [ ] OIDC provider associated (for IRSA)
- [ ] EKS add-ons managed (vpc-cni, coredns, kube-proxy, ebs-csi)
- [ ] Envelope encryption enabled for Secrets (KMS)

### Networking

- [ ] 3 AZ deployment (minimum)
- [ ] Nodes in private subnets only
- [ ] NAT Gateway (HA — one per AZ)
- [ ] VPC endpoints for AWS services
- [ ] Network policies enabled & default-deny
- [ ] Proper subnet tagging for LB discovery

### Security

- [ ] IRSA for all pod-to-AWS interactions (no node-level keys)
- [ ] Pod Security Standards enforced (restricted)
- [ ] Network Policies applied
- [ ] Image scanning on push (ECR)
- [ ] No `latest` tags — pin image digests or versions
- [ ] Secrets in AWS Secrets Manager (not K8s Secrets)
- [ ] Runtime security (Falco/GuardDuty)
- [ ] WAF on public ALBs
- [ ] Kyverno/Gatekeeper policies enforced
- [ ] Container runs as non-root with read-only filesystem
- [ ] `aws-auth` ConfigMap properly configured (or Access Entries)

### Reliability

- [ ] Pod Disruption Budgets (PDB) on all Deployments
- [ ] Resource requests & limits set on every container
- [ ] Liveness, readiness, and startup probes configured
- [ ] HPA configured for variable workloads
- [ ] Cluster Autoscaler or Karpenter running
- [ ] Anti-affinity rules to spread pods across AZs/nodes
- [ ] Velero backups scheduled
- [ ] Multi-AZ for all stateful services (RDS, ElastiCache)

### Observability

- [ ] Container Insights or Prometheus + Grafana
- [ ] Centralized logging (Fluent Bit -> CloudWatch/OpenSearch)
- [ ] Alerting configured (PagerDuty/Slack)
- [ ] Distributed tracing (X-Ray / Jaeger)
- [ ] Dashboard for SLIs/SLOs

### CI/CD

- [ ] GitOps with ArgoCD or Flux
- [ ] Image built and scanned in CI
- [ ] Rolling update strategy with rollback
- [ ] Canary / Blue-Green deployment support

---

## 20. Terraform (IaC) Reference

> **⚠️ WARNING:** Always use Infrastructure as Code for production EKS clusters. Never rely on CLI commands alone.

### Directory Structure

```
terraform/
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── staging/
├── modules/
│   ├── vpc/
│   ├── eks/
│   ├── node-groups/
│   └── addons/
```

### Main Terraform Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "eks/production/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

# main.tf
terraform {
  required_version = ">= 1.7"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.70"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.33"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.16"
    }
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "terraform"
      Project     = "eks-platform"
    }
  }
}

# --- VPC Module ---
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.13"

  name = "${var.cluster_name}-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]

  private_subnets = ["10.0.10.0/24", "10.0.20.0/24", "10.0.30.0/24"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false
  one_nat_gateway_per_az = true
  enable_dns_hostnames   = true
  enable_dns_support     = true

  public_subnet_tags = {
    "kubernetes.io/role/elb"                      = 1
    "kubernetes.io/cluster/${var.cluster_name}"    = "shared"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"              = 1
    "kubernetes.io/cluster/${var.cluster_name}"    = "shared"
    "karpenter.sh/discovery"                       = var.cluster_name
  }
}

# --- EKS Cluster ---
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.24"

  cluster_name    = var.cluster_name
  cluster_version = "1.30"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  enable_irsa = true

  cluster_enabled_log_types = [
    "api", "audit", "authenticator",
    "controllerManager", "scheduler"
  ]

  # Encryption
  cluster_encryption_config = {
    provider_key_arn = aws_kms_key.eks.arn
    resources        = ["secrets"]
  }

  # EKS Add-ons
  cluster_addons = {
    coredns    = { most_recent = true }
    kube-proxy = { most_recent = true }
    vpc-cni = {
      most_recent    = true
      before_compute = true
      configuration_values = jsonencode({
        enableNetworkPolicy = "true"
      })
    }
    aws-ebs-csi-driver = {
      most_recent              = true
      service_account_role_arn = module.ebs_csi_irsa.iam_role_arn
    }
  }

  # Managed Node Groups
  eks_managed_node_groups = {
    system = {
      instance_types = ["m6i.xlarge"]
      min_size       = 2
      max_size       = 5
      desired_size   = 3

      labels = { role = "system" }
      tags   = { NodeGroupRole = "system" }
    }

    application = {
      instance_types = ["m6i.2xlarge"]
      min_size       = 3
      max_size       = 20
      desired_size   = 5

      labels = { role = "application" }
      taints = [{
        key    = "dedicated"
        value  = "application"
        effect = "NO_SCHEDULE"
      }]
    }
  }

  # Access management
  enable_cluster_creator_admin_permissions = true
  access_entries = {
    devops = {
      principal_arn = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/DevOpsRole"
      policy_associations = {
        admin = {
          policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
          access_scope = { type = "cluster" }
        }
      }
    }
  }

  tags = var.tags
}

# --- KMS Key for EKS Secrets Encryption ---
resource "aws_kms_key" "eks" {
  description             = "EKS Secret Encryption Key"
  deletion_window_in_days = 7
  enable_key_rotation     = true
}

# --- Outputs ---
output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_name" {
  value = module.eks.cluster_name
}

output "configure_kubectl" {
  value = "aws eks update-kubeconfig --name ${module.eks.cluster_name} --region ${var.region}"
}
```

```bash
# Deploy
cd terraform/environments/production
terraform init
terraform plan -out=plan.out
terraform apply plan.out
```

---

## Quick Reference — Key Commands

```bash
# Cluster info
eksctl get cluster
kubectl cluster-info
kubectl get nodes -o wide

# Debug a pod
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
kubectl exec -it <pod> -n <ns> -- /bin/sh

# Rolling restart
kubectl rollout restart deployment/<name> -n <ns>
kubectl rollout status deployment/<name> -n <ns>

# Upgrade cluster
eksctl upgrade cluster --name production-cluster --version 1.31 --approve

# Upgrade node groups (managed)
eksctl upgrade nodegroup \
  --cluster production-cluster \
  --name app-nodes \
  --kubernetes-version 1.31

# Drain a node for maintenance
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

---

> **TIP:** Recommended learning path — Start with `eksctl` for initial setup, then migrate everything to Terraform modules for repeatability and version control. Use ArgoCD for GitOps-driven deployments.

---

*Last updated: November 2026*
