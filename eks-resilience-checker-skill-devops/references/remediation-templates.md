# EKS Resilience Remediation Templates

> Actionable remediation commands for each check that can FAIL.
> Info-only checks (A9, A10, A12, C3, D7) are excluded as they are advisory.
> Replace `{placeholder}` values with actual resource names.

---

## Application Checks

### A1: Avoid Running Singleton Pods
**Cost Impact:** Zero — pure K8s resource conversion (Pod to Deployment)
#### Remediation
Convert the singleton pod to a Deployment:
```bash
# Generate a Deployment manifest from the running pod
kubectl get pod {pod-name} -n {namespace} -o json | jq '{apiVersion: "apps/v1", kind: "Deployment", metadata: {name: .metadata.name, namespace: .metadata.namespace}, spec: {replicas: 2, selector: {matchLabels: {app: .metadata.name}}, template: {metadata: {labels: {app: .metadata.name}}, spec: .spec}}}' > /tmp/{pod-name}-deployment.yaml

# Review and apply
kubectl apply -f /tmp/{pod-name}-deployment.yaml

# Delete the original singleton pod
kubectl delete pod {pod-name} -n {namespace}
```
#### Notes
- Review the generated manifest before applying — adjust labels, resource limits, probes, etc.
- Preserve the pod's environment variables, volumes, and service account configuration
- Consider adding health probes and resource limits in the new Deployment spec

---

### A2: Run Multiple Replicas
**Cost Impact:** +1 Pod per workload — doubles CPU/memory; may trigger additional node
#### Remediation
Scale up the workload to at least 2 replicas:
```bash
# Scale a Deployment
kubectl scale deployment {deployment-name} --replicas=2 -n {namespace}

# Scale a StatefulSet
kubectl scale statefulset {statefulset-name} --replicas=2 -n {namespace}

# Or patch the spec directly
kubectl patch deployment {deployment-name} -n {namespace} -p '{"spec":{"replicas":2}}'
```
#### Notes
- Ensure the application supports running multiple replicas (stateless or with proper coordination)
- For StatefulSets, verify the application handles peer discovery correctly
- Check that sufficient cluster resources are available for additional replicas

---

### A3: Use Pod Anti-Affinity
**Cost Impact:** Zero — K8s scheduling configuration only
#### Remediation
Add `podAntiAffinity` to spread replicas across nodes:
```bash
kubectl patch deployment {deployment-name} -n {namespace} --type=strategic -p '{
  "spec": {
    "template": {
      "spec": {
        "affinity": {
          "podAntiAffinity": {
            "preferredDuringSchedulingIgnoredDuringExecution": [{
              "weight": 100,
              "podAffinityTerm": {
                "labelSelector": {
                  "matchExpressions": [{
                    "key": "app",
                    "operator": "In",
                    "values": ["{deployment-name}"]
                  }]
                },
                "topologyKey": "kubernetes.io/hostname"
              }
            }]
          }
        }
      }
    }
  }
}'
```
#### Notes
- Use `preferredDuringSchedulingIgnoredDuringExecution` (soft) to avoid scheduling failures on small clusters
- Use `requiredDuringSchedulingIgnoredDuringExecution` (hard) for critical workloads with enough nodes
- Adjust the label selector key (`app`) to match your actual pod labels
- Consider adding `topologyKey: topology.kubernetes.io/zone` for AZ-level spreading

---

### A4: Use Liveness Probes
**Cost Impact:** Zero — K8s probe configuration only
#### Remediation
Add a liveness probe to each container:
```bash
kubectl patch deployment {deployment-name} -n {namespace} --type=strategic -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "{container-name}",
          "livenessProbe": {
            "httpGet": {
              "path": "/healthz",
              "port": 8080
            },
            "initialDelaySeconds": 15,
            "periodSeconds": 10,
            "timeoutSeconds": 5,
            "failureThreshold": 3
          }
        }]
      }
    }
  }
}'
```
#### Notes
- Choose the probe type that matches your application: `httpGet`, `tcpSocket`, or `exec`
- Set `initialDelaySeconds` high enough for the application to start
- Liveness probes should check deep health (not just port open) — deadlock detection, critical dependency availability
- Avoid making liveness probes depend on external services — this can cause cascading restarts

---

### A5: Use Readiness Probes
**Cost Impact:** Zero — K8s probe configuration only
#### Remediation
Add a readiness probe to each container:
```bash
kubectl patch deployment {deployment-name} -n {namespace} --type=strategic -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "{container-name}",
          "readinessProbe": {
            "httpGet": {
              "path": "/ready",
              "port": 8080
            },
            "initialDelaySeconds": 5,
            "periodSeconds": 5,
            "timeoutSeconds": 3,
            "failureThreshold": 3
          }
        }]
      }
    }
  }
}'
```
#### Notes
- Readiness probes should be different from liveness probes — check if the app is ready to serve traffic
- Include downstream dependency checks (database connections, cache warmup) in readiness
- Use a shorter `periodSeconds` than liveness probes for faster traffic routing decisions
- Readiness probe failure removes the pod from Service endpoints but does NOT restart it

---

### A6: Use Pod Disruption Budgets
**Cost Impact:** Zero — K8s PDB configuration only
#### Remediation
Create a PDB for the workload:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {workload-name}-pdb
  namespace: {namespace}
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: {workload-name}
EOF
```
#### Notes
- Use `minAvailable` (absolute count or percentage) OR `maxUnavailable`, not both
- For 2-replica Deployments: `minAvailable: 1` or `maxUnavailable: 1`
- For 3+ replica Deployments: `maxUnavailable: 1` or `minAvailable: "50%"`
- Match the PDB selector labels exactly with the workload's pod labels
- PDBs only apply to voluntary disruptions (node drain, cluster upgrades), not involuntary ones (node crash)

---

### A7: Run Kubernetes Metrics Server
**Cost Impact:** ~0.5 vCPU + 256MB memory for metrics-server Pod
#### Remediation
Install metrics-server:
```bash
# Install via kubectl
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Or install as EKS managed add-on (recommended)
CLUSTER_NAME="{cluster-name}"
REGION="ap-northeast-1"
aws eks create-addon --cluster-name "$CLUSTER_NAME" --addon-name metrics-server --region "$REGION"

# Verify installation
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```
#### Notes
- EKS managed add-on is preferred for automatic updates and compatibility
- Verify that the metrics API is accessible: `kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes`
- Metrics-server requires proper network connectivity to kubelets (port 10250)

---

### A8: Use Horizontal Pod Autoscaler
**Cost Impact:** HPA free; autoscaled Pods may increase compute cost
#### Remediation
Create an HPA for the workload:
```bash
# Simple CPU-based HPA
kubectl autoscale deployment {deployment-name} -n {namespace} --min=2 --max=10 --cpu-percent=70

# Or create a detailed HPA manifest
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {deployment-name}-hpa
  namespace: {namespace}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {deployment-name}
  minReplicas: 2
  maxReplicas: 10
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
    scaleDown:
      stabilizationWindowSeconds: 300
EOF
```
#### Notes
- Requires metrics-server to be running (check A7 first)
- Set `minReplicas >= 2` for high availability
- Add `scaleDown.stabilizationWindowSeconds` to prevent flapping
- Ensure resource requests are set on containers (HPA uses request-based utilization)

---

### A11: Use PreStop Hooks
**Cost Impact:** Zero — K8s lifecycle configuration only
#### Remediation
Add a preStop hook to gracefully handle termination:
```bash
kubectl patch deployment {deployment-name} -n {namespace} --type=strategic -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "{container-name}",
          "lifecycle": {
            "preStop": {
              "exec": {
                "command": ["/bin/sh", "-c", "sleep 10"]
              }
            }
          }
        }]
      }
    }
  }
}'
```
#### Notes
- The `sleep` approach gives in-flight requests time to complete before SIGTERM
- For web servers: use a command that triggers graceful shutdown (e.g., `nginx -s quit`)
- Ensure `terminationGracePeriodSeconds` > preStop duration (default is 30s)
- DaemonSets are intentionally excluded from this check — they run on every node and are system services

---

### A13: Monitor Your Applications
**Cost Impact:** CloudWatch Container Insights: per-metric + log volume pricing; Prometheus: ~2 vCPU + 8GB
#### Remediation
Install a monitoring solution. Recommended: Prometheus stack via Helm:
```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.enabled=true \
  --set alertmanager.enabled=true

# Or enable CloudWatch Container Insights
aws eks create-addon --cluster-name {cluster-name} --addon-name amazon-cloudwatch-observability --region ap-northeast-1
```
#### Notes
- `kube-prometheus-stack` includes Prometheus, Grafana, Alertmanager, and node-exporter
- CloudWatch Container Insights is simpler to set up but has higher cost at scale
- Third-party options (Datadog, New Relic, Dynatrace) are also valid

---

### A14: Use Centralized Logging
**Cost Impact:** Fluent Bit DaemonSet: ~0.5 vCPU + 256MB per node; CW Logs ~\$0.50/GB ingested
#### Remediation
Install a centralized logging solution. Recommended: Fluent Bit to CloudWatch Logs:
```bash
# Install Fluent Bit as EKS add-on
aws eks create-addon --cluster-name {cluster-name} --addon-name aws-for-fluent-bit --region ap-northeast-1

# Or install via Helm
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm install fluent-bit fluent/fluent-bit \
  --namespace logging --create-namespace \
  --set output.cloudwatch.enabled=true \
  --set output.cloudwatch.region=ap-northeast-1 \
  --set output.cloudwatch.logGroupName="/eks/{cluster-name}/application"
```
#### Notes
- Ensure the Fluent Bit service account has IAM permissions to write to CloudWatch Logs
- For EKS, use IRSA (IAM Roles for Service Accounts) for secure credential management
- Alternative stacks: EFK (Elasticsearch + Fluentd + Kibana), PLG (Promtail + Loki + Grafana)

---

## Control Plane Checks

### C1: Monitor Control Plane Logs
**Cost Impact:** CloudWatch Logs: ~\$0.50/GB ingested (control plane ~1-5 GB/month)
#### Remediation
Enable EKS control plane logging:
```bash
CLUSTER_NAME="{cluster-name}"
REGION="ap-northeast-1"

aws eks update-cluster-config --name "$CLUSTER_NAME" --region "$REGION" \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Verify the update
aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" --query 'cluster.logging'
```
#### Notes
- Enabling all 5 log types is recommended: `api`, `audit`, `authenticator`, `controllerManager`, `scheduler`
- Logs go to CloudWatch Logs group `/aws/eks/{cluster-name}/cluster`
- Be aware of CloudWatch Logs costs — consider setting a retention policy
- The update is asynchronous; check cluster status until `ACTIVE`

---

### C2: Cluster Authentication
**Cost Impact:** Zero — authentication configuration only
#### Remediation
Configure EKS Access Entries (recommended over aws-auth ConfigMap):
```bash
CLUSTER_NAME="{cluster-name}"
REGION="ap-northeast-1"

# Create an access entry for an IAM role
aws eks create-access-entry --cluster-name "$CLUSTER_NAME" --region "$REGION" \
  --principal-arn "arn:aws:iam::{account-id}:role/{role-name}" \
  --type STANDARD

# Associate an access policy
aws eks associate-access-policy --cluster-name "$CLUSTER_NAME" --region "$REGION" \
  --principal-arn "arn:aws:iam::{account-id}:role/{role-name}" \
  --policy-arn "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy" \
  --access-scope '{"type":"cluster"}'
```
#### Notes
- EKS Access Entries (API mode) is the modern, recommended approach
- Migrate from `aws-auth` ConfigMap to Access Entries when possible
- Use namespace-scoped access policies for least-privilege access
- Test authentication changes carefully — misconfigurations can lock you out

---

### C4: EKS Control Plane Endpoint Access Control
**Cost Impact:** Zero — endpoint access configuration only
#### Remediation
Restrict API server endpoint access:
```bash
CLUSTER_NAME="{cluster-name}"
REGION="ap-northeast-1"

# Option 1: Private-only access (most secure)
aws eks update-cluster-config --name "$CLUSTER_NAME" --region "$REGION" \
  --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true

# Option 2: Public + private with CIDR restriction
aws eks update-cluster-config --name "$CLUSTER_NAME" --region "$REGION" \
  --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs="{your-cidr}/32"
```
#### Notes
- Private-only access requires VPN, Direct Connect, or bastion host for `kubectl` access
- When using public + private, restrict `publicAccessCidrs` to your office/VPN CIDR blocks
- Never use `0.0.0.0/0` for production clusters
- The update is asynchronous; cluster will be in `UPDATING` status temporarily

---

### C5: Avoid Catch-All Admission Webhooks
**Cost Impact:** Zero — webhook scope configuration only
#### Remediation
Add namespace selectors and narrow the scope of webhook rules:
```bash
# Patch a MutatingWebhookConfiguration to add a namespace selector
kubectl patch mutatingwebhookconfiguration {webhook-name} --type=json -p '[{
  "op": "add",
  "path": "/webhooks/0/namespaceSelector",
  "value": {
    "matchExpressions": [{
      "key": "kubernetes.io/metadata.name",
      "operator": "NotIn",
      "values": ["kube-system", "kube-public", "kube-node-lease"]
    }]
  }
}]'

# Narrow wildcard rules to specific resources
kubectl patch validatingwebhookconfiguration {webhook-name} --type=json -p '[{
  "op": "replace",
  "path": "/webhooks/0/rules",
  "value": [{
    "apiGroups": ["apps"],
    "apiVersions": ["v1"],
    "resources": ["deployments", "statefulsets"],
    "operations": ["CREATE", "UPDATE"],
    "scope": "Namespaced"
  }]
}]'
```
#### Notes
- Always add `namespaceSelector` to exclude system namespaces from webhook processing
- Replace wildcard `*` in apiGroups/resources with specific values
- Set `failurePolicy: Ignore` for non-critical webhooks to prevent cluster-wide blocking
- Test webhook changes in a non-production environment first

---

## Data Plane Checks

### D1: Use Kubernetes Cluster Autoscaler or Karpenter
**Cost Impact:** Karpenter free; CA: ~0.5 vCPU; auto-scaling increases EC2 spend
#### Remediation
Install Karpenter (recommended) or Cluster Autoscaler:
```bash
# Option 1: Install Karpenter (recommended for new clusters)
# See https://karpenter.sh/docs/getting-started/ for full setup
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm install karpenter karpenter/karpenter \
  --namespace karpenter --create-namespace \
  --set settings.clusterName={cluster-name} \
  --set settings.clusterEndpoint=$(aws eks describe-cluster --name {cluster-name} --query 'cluster.endpoint' --output text) \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::{account-id}:role/KarpenterControllerRole"

# Create a default NodePool
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: "karpenter.k8s.aws/instance-category"
          operator: In
          values: ["c", "m", "r"]
        - key: "topology.kubernetes.io/zone"
          operator: In
          values: ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: "100"
    memory: 400Gi
EOF

# Option 2: Install Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName={cluster-name} \
  --set awsRegion=ap-northeast-1
```
#### Notes
- Karpenter is AWS-native and generally faster at scaling decisions than Cluster Autoscaler
- Both require IAM permissions for EC2 Auto Scaling / Fleet management
- Use IRSA (IAM Roles for Service Accounts) for secure credential management
- Ensure node groups or NodePools span multiple AZs

---

### D2: Worker Nodes Spread Across Multiple AZs
**Cost Impact:** May require additional nodes in underrepresented AZs
#### Remediation
Configure node groups across multiple AZs:
```bash
CLUSTER_NAME="{cluster-name}"
REGION="ap-northeast-1"

# For managed node groups — update subnets to span multiple AZs
aws eks update-nodegroup-config --cluster-name "$CLUSTER_NAME" --region "$REGION" \
  --nodegroup-name "{nodegroup-name}" \
  --scaling-config minSize=3,maxSize=9,desiredSize=6

# Create a new multi-AZ node group if existing one is single-AZ
aws eks create-nodegroup --cluster-name "$CLUSTER_NAME" --region "$REGION" \
  --nodegroup-name "{nodegroup-name}-multi-az" \
  --subnets "subnet-az-a" "subnet-az-c" "subnet-az-d" \
  --instance-types "m5.large" \
  --scaling-config minSize=3,maxSize=9,desiredSize=6

# For Karpenter — ensure NodePool requirements include multiple AZs
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: multi-az
spec:
  template:
    spec:
      requirements:
        - key: "topology.kubernetes.io/zone"
          operator: In
          values: ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
EOF
```
#### Notes
- Use at least 3 AZs for production clusters
- Ensure subnets in each AZ have sufficient IP addresses
- Set `desiredSize` to a multiple of the number of AZs for balanced distribution
- Existing single-AZ node groups cannot be converted — create new multi-AZ groups and migrate

---

### D3: Configure Resource Requests/Limits
**Cost Impact:** Zero — may expose need for more capacity if requests were previously unset
#### Remediation
Add resource specifications to containers:
```bash
kubectl patch deployment {deployment-name} -n {namespace} --type=strategic -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "{container-name}",
          "resources": {
            "requests": {
              "cpu": "100m",
              "memory": "128Mi"
            },
            "limits": {
              "cpu": "500m",
              "memory": "512Mi"
            }
          }
        }]
      }
    }
  }
}'
```
#### Notes
- Start with conservative values and adjust based on actual usage (`kubectl top pods`)
- Requests determine scheduling — set to the typical resource usage
- Limits prevent resource abuse — set to the maximum acceptable burst usage
- Use VPA (check A10) recommendations to right-size resource values
- CPU limits are controversial — some teams only set CPU requests. Memory limits are generally always recommended

---

### D4: Namespace ResourceQuotas
**Cost Impact:** Zero — K8s quota configuration only
#### Remediation
Create a ResourceQuota for the namespace:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
  namespace: {namespace}
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
EOF
```
#### Notes
- Adjust quota values based on the namespace's expected workload
- ResourceQuotas require that all pods in the namespace have resource requests/limits set
- Consider creating quotas for different priority classes
- Monitor quota usage with `kubectl describe resourcequota -n {namespace}`

---

### D5: Namespace LimitRanges
**Cost Impact:** Zero — K8s LimitRange configuration only
#### Remediation
Create a LimitRange for the namespace:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: {namespace}
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
EOF
```
#### Notes
- LimitRange provides default resource values for containers that don't specify them
- Complements ResourceQuotas — LimitRange sets per-pod defaults, ResourceQuota sets per-namespace totals
- The `default` values are applied to containers without explicit limits
- The `defaultRequest` values are applied to containers without explicit requests

---

### D6: Monitor CoreDNS Metrics
**Cost Impact:** Zero — metrics endpoint configuration only
#### Remediation
Set up CoreDNS metrics collection:
```bash
# If using Prometheus Operator — create a ServiceMonitor
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: coredns
  namespace: kube-system
  labels:
    app: coredns
spec:
  selector:
    matchLabels:
      k8s-app: kube-dns
  endpoints:
  - port: metrics
    interval: 15s
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
EOF

# If using Prometheus scrape annotations
kubectl annotate service kube-dns -n kube-system \
  prometheus.io/scrape="true" \
  prometheus.io/port="9153"
```
#### Notes
- CoreDNS exposes metrics on port 9153 by default
- Key metrics to monitor: `coredns_dns_requests_total`, `coredns_dns_responses_rcode_count_total`, `coredns_dns_request_duration_seconds`
- Alert on elevated NXDOMAIN or SERVFAIL response codes
- Monitor DNS latency — high latency affects all service-to-service communication
