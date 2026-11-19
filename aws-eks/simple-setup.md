# ⚡ EKS Simple Setup Guide

A minimal, copy-paste guide to get an EKS cluster running in ~15 minutes.

---

## Step 1 — Install Tools

```bash
# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# kubectl
curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# eksctl
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xzf eksctl_Linux_amd64.tar.gz -C /tmp && sudo mv /tmp/eksctl /usr/local/bin

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## Step 2 — Configure AWS

```bash
aws configure
# Enter: Access Key, Secret Key, Region (e.g. us-east-1), Output (json)

# Verify
aws sts get-caller-identity
```

---

## Step 3 — Create the Cluster

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --version 1.30 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed
```

> ☕ This takes ~15 minutes. It creates the VPC, subnets, NAT gateway, IAM roles, cluster, and node group automatically.

---

## Step 4 — Verify

```bash
# kubeconfig is auto-updated by eksctl, but you can do it manually:
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Check cluster
kubectl cluster-info
kubectl get nodes
```

Expected output:
```
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-xx-xx.ec2.internal   Ready    <none>   2m    v1.30.x
ip-192-168-xx-xx.ec2.internal   Ready    <none>   2m    v1.30.x
```

---

## Step 5 — Deploy a Sample App

```yaml
# Save as app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f app.yaml

# Wait for the LoadBalancer URL
kubectl get svc nginx -w
```

Open the `EXTERNAL-IP` in your browser — you'll see the Nginx welcome page.

---

## Step 6 — Install EBS Storage (Optional)

```bash
# Enable OIDC (needed for IRSA)
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# Install EBS CSI driver add-on
eksctl create addon --cluster my-cluster --name aws-ebs-csi-driver \
  --service-account-role-arn $(eksctl create iamserviceaccount \
    --cluster my-cluster \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve \
    --role-only \
    --role-name AmazonEKS_EBS_CSI_DriverRole 2>&1 | grep -oP 'arn:aws:iam::\d+:role/\S+')
```

```yaml
# gp3-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f gp3-storageclass.yaml
```

---

## Step 7 — Basic Autoscaling (Optional)

### HPA (Pod-level scaling)

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Create HPA
kubectl autoscale deployment nginx --cpu-percent=70 --min=2 --max=10
kubectl get hpa
```

### Cluster Autoscaler (Node-level scaling)

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1
```

---

## Step 8 — Useful kubectl Commands

```bash
# Pods
kubectl get pods                        # List pods
kubectl get pods -A                     # All namespaces
kubectl describe pod <name>             # Debug a pod
kubectl logs <name>                     # View logs
kubectl logs <name> -f                  # Stream logs
kubectl exec -it <name> -- /bin/sh      # Shell into pod

# Deployments
kubectl get deploy                      # List deployments
kubectl scale deploy nginx --replicas=5 # Scale
kubectl rollout restart deploy nginx    # Restart
kubectl rollout undo deploy nginx       # Rollback

# Services & Networking
kubectl get svc                         # List services
kubectl get ingress                     # List ingress

# Nodes
kubectl get nodes -o wide              # Node details
kubectl top nodes                      # Node resource usage
kubectl top pods                       # Pod resource usage

# Namespaces
kubectl create ns my-app               # Create namespace
kubectl get all -n my-app              # All resources in ns
```

---

## Cleanup

```bash
# Delete the app
kubectl delete -f app.yaml

# Delete the entire cluster (removes everything)
eksctl delete cluster --name my-cluster --region us-east-1
```

> ⚠️ This deletes all resources including the VPC, nodes, and load balancers. Make sure you want to do this.

---

## What's Next?

When you're ready for production, check the full guide:
→ [EKS-Setup-Guide.md](./EKS-Setup-Guide.md)

Key things to add for production:
- Private subnets for nodes
- IRSA (IAM Roles for Service Accounts)
- Network Policies
- Ingress with ALB Controller + TLS
- Prometheus/Grafana monitoring
- Velero backups
- GitOps with ArgoCD
- Terraform for infrastructure-as-code
