# EKS Authentication Setup

Two methods to authenticate kubectl with an EKS cluster.

## Method 1: IAM-based kubeconfig (Recommended)

Uses `aws eks update-kubeconfig` to generate a kubeconfig that obtains tokens via `aws eks get-token` on each request.

```bash
# Generate kubeconfig for the target cluster
aws eks update-kubeconfig --name {CLUSTER_NAME} --region {REGION}

# Verify access
kubectl get nodes
```

If the user already has a kubeconfig (e.g., provided by a cluster admin):

```bash
# Point to an existing kubeconfig
export KUBECONFIG=/path/to/admin-kubeconfig
kubectl get nodes

# Or specify per-command
kubectl --kubeconfig /path/to/admin-kubeconfig get nodes
```

> **In Step 1**, ask the user: "Do you have an existing kubeconfig, or should I generate one with `aws eks update-kubeconfig`?"

**Requirements:**
- AWS CLI installed and configured with valid credentials (if generating kubeconfig)
- IAM identity must have `eks:DescribeCluster` permission
- IAM identity must be mapped in the cluster's access configuration (EKS Access Entries or `aws-auth` ConfigMap)

**Pros:** Uses existing AWS credentials, standard EKS workflow, automatic token refresh
**Cons:** Requires AWS CLI + IAM credentials on the machine running the assessment

## Method 2: Static Service Account Token (For restricted environments)

Creates a Kubernetes ServiceAccount with read-only permissions and generates a self-contained kubeconfig. **No AWS CLI or IAM credentials required at runtime.**

```bash
# 1. Create ServiceAccount with read-only RBAC
kubectl create serviceaccount eks-resilience-checker -n kube-system

# 2. Create ClusterRoleBinding (read-only access)
kubectl create clusterrolebinding eks-resilience-checker-readonly \
  --clusterrole=view \
  --serviceaccount=kube-system:eks-resilience-checker

# 3. Generate token (valid for 1 year)
TOKEN=$(kubectl create token eks-resilience-checker -n kube-system --duration=8760h)

# 4. Get cluster endpoint and CA
ENDPOINT=$(aws eks describe-cluster --name {CLUSTER_NAME} --query 'cluster.endpoint' --output text)
CA_DATA=$(aws eks describe-cluster --name {CLUSTER_NAME} --query 'cluster.certificateAuthority.data' --output text)

# 5. Generate self-contained kubeconfig
kubectl config set-cluster eks-check --server=$ENDPOINT --certificate-authority-data=$CA_DATA --embed-certs=true --kubeconfig=./eks-resilience-kubeconfig
kubectl config set-credentials eks-checker --token=$TOKEN --kubeconfig=./eks-resilience-kubeconfig
kubectl config set-context eks-check --cluster=eks-check --user=eks-checker --kubeconfig=./eks-resilience-kubeconfig
kubectl config use-context eks-check --kubeconfig=./eks-resilience-kubeconfig

# 6. Use the generated kubeconfig
export KUBECONFIG=./eks-resilience-kubeconfig
kubectl get nodes
```

**Pros:** Portable, no AWS dependency at runtime, least-privilege (read-only), works in CI/CD pipelines
**Cons:** Token has fixed expiry (default 1 year), needs renewal, requires initial setup with cluster admin access

## Recommendation

- Use **Method 1** for interactive assessments
- Use **Method 2** for CI/CD pipelines, automated periodic checks, or environments where AWS CLI is not available
