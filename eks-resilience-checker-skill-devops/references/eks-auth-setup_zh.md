# EKS 认证配置

kubectl 连接 EKS 集群有两种方式。

## 方式一：IAM kubeconfig（推荐）

使用 `aws eks update-kubeconfig` 生成 kubeconfig，每次请求时通过 `aws eks get-token` 获取令牌。

```bash
# 为目标集群生成 kubeconfig
aws eks update-kubeconfig --name {CLUSTER_NAME} --region {REGION}

# 验证访问
kubectl get nodes
```

如果用户已有 kubeconfig（如集群管理员提供的）：

```bash
# 指向已有 kubeconfig
export KUBECONFIG=/path/to/admin-kubeconfig
kubectl get nodes

# 或在命令中指定
kubectl --kubeconfig /path/to/admin-kubeconfig get nodes
```

> **在第 1 步中**，询问用户："你是否已有 kubeconfig，还是需要我用 `aws eks update-kubeconfig` 生成一个？"

**要求：**
- AWS CLI 已安装并配置有效凭证（如需生成 kubeconfig）
- IAM 身份必须拥有 `eks:DescribeCluster` 权限
- IAM 身份必须在集群访问配置中映射（EKS Access Entries 或 `aws-auth` ConfigMap）

**优点：** 使用现有 AWS 凭证，标准 EKS 工作流，自动刷新令牌
**缺点：** 需要运行评估的机器上有 AWS CLI + IAM 凭证

## 方式二：静态 Service Account 令牌（受限环境）

创建具有只读权限的 Kubernetes ServiceAccount，生成自包含的 kubeconfig。**运行时无需 AWS CLI 或 IAM 凭证。**

```bash
# 1. 创建具有只读 RBAC 的 ServiceAccount
kubectl create serviceaccount eks-resilience-checker -n kube-system

# 2. 创建 ClusterRoleBinding（只读访问）
kubectl create clusterrolebinding eks-resilience-checker-readonly \
  --clusterrole=view \
  --serviceaccount=kube-system:eks-resilience-checker

# 3. 生成令牌（有效期 1 年）
TOKEN=$(kubectl create token eks-resilience-checker -n kube-system --duration=8760h)

# 4. 获取集群端点和 CA
ENDPOINT=$(aws eks describe-cluster --name {CLUSTER_NAME} --query 'cluster.endpoint' --output text)
CA_DATA=$(aws eks describe-cluster --name {CLUSTER_NAME} --query 'cluster.certificateAuthority.data' --output text)

# 5. 生成自包含 kubeconfig
kubectl config set-cluster eks-check --server=$ENDPOINT --certificate-authority-data=$CA_DATA --embed-certs=true --kubeconfig=./eks-resilience-kubeconfig
kubectl config set-credentials eks-checker --token=$TOKEN --kubeconfig=./eks-resilience-kubeconfig
kubectl config set-context eks-check --cluster=eks-check --user=eks-checker --kubeconfig=./eks-resilience-kubeconfig
kubectl config use-context eks-check --kubeconfig=./eks-resilience-kubeconfig

# 6. 使用生成的 kubeconfig
export KUBECONFIG=./eks-resilience-kubeconfig
kubectl get nodes
```

**优点：** 可移植，运行时无 AWS 依赖，最小权限（只读），适合 CI/CD 流水线
**缺点：** 令牌有固定到期时间（默认 1 年），需要续期，初始配置需要集群管理员权限

## 建议

- 交互式评估使用**方式一**
- CI/CD 流水线、自动化定期检查或无 AWS CLI 环境使用**方式二**
