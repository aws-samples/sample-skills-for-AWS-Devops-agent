# EKS Resilience Check Commands

> **Scope**: This file covers **how** to execute each check (exact commands + PASS/FAIL criteria).
> For **what** each check does and **why** it matters, see [EKS-Resiliency-Checkpoints.md](EKS-Resiliency-Checkpoints.md).

> Exact runnable commands, PASS criteria, and severity for each of the 26 checks.
> System namespaces excluded: `kube-system`, `kube-public`, `kube-node-lease`.

---

## Application Checks (A1–A14)

### A1: Avoid Running Singleton Pods
**Severity**: 🔴 Critical

#### Check Command
```bash
kubectl get pods --all-namespaces -o json | jq '[.items[] | select((.metadata.ownerReferences // []) | length == 0) | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name}]'
```

#### PASS Criteria
Empty array `[]` — no singleton pods found in non-system namespaces.

---

### A2: Run Multiple Replicas
**Severity**: 🔴 Critical

#### Check Command
```bash
# Check Deployments
kubectl get deployments --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | select(.spec.replicas == 1) | {namespace: .metadata.namespace, name: .metadata.name, replicas: .spec.replicas}]'

# Check StatefulSets
kubectl get statefulsets --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | select(.spec.replicas == 1) | {namespace: .metadata.namespace, name: .metadata.name, replicas: .spec.replicas}]'
```

#### PASS Criteria
Both commands return `[]` — all Deployments and StatefulSets have `replicas > 1`.

---

### A3: Use Pod Anti-Affinity
**Severity**: 🟡 Warning

#### Check Command
```bash
kubectl get deployments --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | select(.spec.replicas > 1) | select(.spec.template.spec.affinity.podAntiAffinity == null) | {namespace: .metadata.namespace, name: .metadata.name, replicas: .spec.replicas}]'
```

#### PASS Criteria
Empty array `[]` — all multi-replica Deployments have `podAntiAffinity` configured.

---

### A4: Use Liveness Probes
**Severity**: 🔴 Critical

#### Check Command
```bash
# Check Deployments
kubectl get deployments --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_probe: [.spec.template.spec.containers[] | select(.livenessProbe == null) | .name]} | select(.containers_missing_probe | length > 0)]'

# Check StatefulSets
kubectl get statefulsets --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_probe: [.spec.template.spec.containers[] | select(.livenessProbe == null) | .name]} | select(.containers_missing_probe | length > 0)]'

# Check DaemonSets
kubectl get daemonsets --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_probe: [.spec.template.spec.containers[] | select(.livenessProbe == null) | .name]} | select(.containers_missing_probe | length > 0)]'
```

#### PASS Criteria
All three commands return `[]` — every container in every workload has a `livenessProbe`.

---

### A5: Use Readiness Probes
**Severity**: 🔴 Critical

#### Check Command
```bash
# Check Deployments
kubectl get deployments --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_probe: [.spec.template.spec.containers[] | select(.readinessProbe == null) | .name]} | select(.containers_missing_probe | length > 0)]'

# Check StatefulSets
kubectl get statefulsets --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_probe: [.spec.template.spec.containers[] | select(.readinessProbe == null) | .name]} | select(.containers_missing_probe | length > 0)]'

# Check DaemonSets
kubectl get daemonsets --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_probe: [.spec.template.spec.containers[] | select(.readinessProbe == null) | .name]} | select(.containers_missing_probe | length > 0)]'
```

#### PASS Criteria
All three commands return `[]` — every container has a `readinessProbe`.

---

### A6: Use Pod Disruption Budgets
**Severity**: 🟡 Warning

#### Check Command
```bash
# Step 1: List all PDBs and their selectors
kubectl get pdb --all-namespaces -o json | jq '[.items[] | {namespace: .metadata.namespace, name: .metadata.name, selector: .spec.selector.matchLabels}]'

# Step 2: Find multi-replica Deployments without a matching PDB
kubectl get deployments --all-namespaces -o json | jq --argjson pdbs "$(kubectl get pdb --all-namespaces -o json | jq '[.items[] | {namespace: .metadata.namespace, selector: .spec.selector.matchLabels}]')" '[.items[] | select(.metadata.namespace | test("^kube-") | not) | select(.spec.replicas > 1) | . as $deploy | {namespace: .metadata.namespace, name: .metadata.name} | select(. as $d | $pdbs | map(select(.namespace == $d.namespace)) | length == 0)]'

# Step 3: Find all StatefulSets without a matching PDB
kubectl get statefulsets --all-namespaces -o json | jq --argjson pdbs "$(kubectl get pdb --all-namespaces -o json | jq '[.items[] | {namespace: .metadata.namespace, selector: .spec.selector.matchLabels}]')" '[.items[] | select(.metadata.namespace | test("^kube-") | not) | . as $sts | {namespace: .metadata.namespace, name: .metadata.name} | select(. as $d | $pdbs | map(select(.namespace == $d.namespace)) | length == 0)]'
```

#### PASS Criteria
Steps 2 and 3 return `[]` — all critical workloads (multi-replica Deployments and all StatefulSets) have corresponding PDBs.

---

### A7: Run Kubernetes Metrics Server
**Severity**: 🟡 Warning

#### Check Command
```bash
# Check metrics-server deployment
kubectl get deployment metrics-server -n kube-system -o json 2>/dev/null | jq '{name: .metadata.name, available_replicas: .status.availableReplicas}'

# Verify metrics API is accessible
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" 2>/dev/null | jq '.items | length'
```

#### PASS Criteria
- Deployment exists with `availableReplicas >= 1`
- Metrics API returns node count > 0

---

### A8: Use Horizontal Pod Autoscaler
**Severity**: 🟡 Warning

#### Check Command
```bash
# List all HPAs
kubectl get hpa --all-namespaces -o json | jq '[.items[] | {namespace: .metadata.namespace, name: .metadata.name, target: .spec.scaleTargetRef.name}]'

# Find multi-replica workloads without HPA
kubectl get deployments --all-namespaces -o json | jq --argjson hpas "$(kubectl get hpa --all-namespaces -o json | jq '[.items[] | {namespace: .metadata.namespace, target: .spec.scaleTargetRef.name}]')" '[.items[] | select(.metadata.namespace | test("^kube-") | not) | select(.spec.replicas > 1) | {namespace: .metadata.namespace, name: .metadata.name} | select(. as $d | $hpas | map(select(.namespace == $d.namespace and .target == $d.name)) | length == 0)]'
```

#### PASS Criteria
Second command returns `[]` — all multi-replica workloads have an HPA.

---

### A9: Use Custom Metrics Scaling
**Severity**: 🟢 Info

#### Check Command
```bash
# Check custom metrics API
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" 2>/dev/null | jq '.resources | length'

# Check external metrics API
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" 2>/dev/null | jq '.resources | length'

# Check for Prometheus Adapter
kubectl get deployment -n kube-system -l app=prometheus-adapter -o json 2>/dev/null | jq '.items | length'

# Check for KEDA
kubectl get deployment -n keda -l app=keda-operator -o json 2>/dev/null | jq '.items | length'
kubectl get crd scaledobjects.keda.sh 2>/dev/null

# Find HPAs using custom or external metrics
kubectl get hpa --all-namespaces -o json | jq '[.items[] | select(.spec.metrics[]? | .type == "Pods" or .type == "Object" or .type == "External") | {namespace: .metadata.namespace, name: .metadata.name}]'
```

#### PASS Criteria
At least one of the following is true:
- Custom metrics API is available
- Prometheus Adapter is deployed
- KEDA is installed
- HPAs with custom/external metrics exist

---

### A10: Use Vertical Pod Autoscaler
**Severity**: 🟢 Info

#### Check Command
```bash
# Check VPA CRD
kubectl get crd verticalpodautoscalers.autoscaling.k8s.io 2>/dev/null

# Check VPA controller components
kubectl get deployment -n kube-system -l app=vpa-recommender -o json 2>/dev/null | jq '.items | length'
kubectl get deployment -n kube-system -l app=vpa-updater -o json 2>/dev/null | jq '.items | length'
kubectl get deployment -n kube-system -l app=vpa-admission-controller -o json 2>/dev/null | jq '.items | length'

# List existing VPA resources
kubectl get vpa --all-namespaces -o json 2>/dev/null | jq '[.items[] | {namespace: .metadata.namespace, name: .metadata.name, target: .spec.targetRef.name, mode: .spec.updatePolicy.updateMode}]'

# Check for Goldilocks
kubectl get deployment -n goldilocks -l app.kubernetes.io/name=goldilocks -o json 2>/dev/null | jq '.items | length'
```

#### PASS Criteria
VPA CRD exists AND at least one VPA resource is configured, or Goldilocks is deployed.

---

### A11: Use PreStop Hooks
**Severity**: 🟡 Warning

#### Check Command
```bash
# Check Deployments (DaemonSets intentionally excluded)
kubectl get deployments --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_hook: [.spec.template.spec.containers[] | select(.lifecycle.preStop == null) | .name]} | select(.containers_missing_hook | length > 0)]'

# Check StatefulSets
kubectl get statefulsets --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_hook: [.spec.template.spec.containers[] | select(.lifecycle.preStop == null) | .name]} | select(.containers_missing_hook | length > 0)]'
```

#### PASS Criteria
Both commands return `[]` — all containers in Deployments and StatefulSets have `preStop` hooks.

---

### A12: Use a Service Mesh
**Severity**: 🟢 Info

#### Check Command
```bash
# Check for Istio
kubectl get namespace istio-system 2>/dev/null && echo "Istio namespace found"
kubectl get crd virtualservices.networking.istio.io 2>/dev/null && echo "Istio CRDs found"

# Check for Linkerd
kubectl get namespace linkerd 2>/dev/null && echo "Linkerd namespace found"
kubectl get crd serviceprofiles.linkerd.io 2>/dev/null && echo "Linkerd CRDs found"

# Check for Consul
kubectl get namespace consul 2>/dev/null && echo "Consul namespace found"

# Check for sidecar proxies in application pods
kubectl get pods --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | select(.spec.containers | length > 1) | select(.spec.containers[].name | test("istio-proxy|linkerd-proxy|envoy-sidecar|consul-sidecar")) | {namespace: .metadata.namespace, name: .metadata.name}] | length'
```

#### PASS Criteria
Any one of the service mesh namespaces/CRDs exists, or sidecar proxies are detected.

---

### A13: Monitor Your Applications
**Severity**: 🟡 Warning

#### Check Command
```bash
# Check for Prometheus stack
kubectl get deployment -n monitoring -l app.kubernetes.io/name=prometheus -o json 2>/dev/null | jq '.items | length'
kubectl get crd prometheuses.monitoring.coreos.com 2>/dev/null && echo "Prometheus Operator CRD found"

# Check for CloudWatch Container Insights
kubectl get daemonset cloudwatch-agent -n amazon-cloudwatch 2>/dev/null && echo "CloudWatch agent found"
kubectl get daemonset fluent-bit -n amazon-cloudwatch 2>/dev/null && echo "Container Insights Fluent Bit found"

# Check for third-party monitoring
kubectl get pods --all-namespaces -o json | jq '[.items[] | select(.metadata.name | test("datadog|newrelic|dynatrace")) | {namespace: .metadata.namespace, name: .metadata.name}]'
```

#### PASS Criteria
At least one monitoring solution detected (Prometheus, CloudWatch Container Insights, or third-party agent).

---

### A14: Use Centralized Logging
**Severity**: 🟡 Warning

#### Check Command
```bash
# Check for Fluent Bit / Fluentd
kubectl get daemonset --all-namespaces -o json | jq '[.items[] | select(.metadata.name | test("fluent-bit|fluentd|fluent")) | {namespace: .metadata.namespace, name: .metadata.name}]'

# Check for CloudWatch Logs agent
kubectl get daemonset --all-namespaces -o json | jq '[.items[] | select(.metadata.name | test("cloudwatch")) | {namespace: .metadata.namespace, name: .metadata.name}]'

# Check for Elasticsearch / OpenSearch
kubectl get deployment --all-namespaces -o json | jq '[.items[] | select(.metadata.name | test("elasticsearch|opensearch|kibana")) | {namespace: .metadata.namespace, name: .metadata.name}]'

# Check for Loki
kubectl get deployment --all-namespaces -o json | jq '[.items[] | select(.metadata.name | test("loki")) | {namespace: .metadata.namespace, name: .metadata.name}]'
```

#### PASS Criteria
At least one logging solution detected (Fluent Bit/Fluentd, CloudWatch Logs, Elasticsearch/OpenSearch, or Loki).

---

## Control Plane Checks (C1–C5)

### C1: Monitor Control Plane Logs
**Severity**: 🟡 Warning

#### Check Command
```bash
CLUSTER_NAME="<cluster-name>"
REGION="ap-northeast-1"

aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" --query 'cluster.logging.clusterLogging[?enabled==`true`].types[]' --output json
```

#### PASS Criteria
Output includes `"api"` in the enabled log types list (e.g., `["api", "audit", "authenticator"]`).

---

### C2: Cluster Authentication
**Severity**: 🟡 Warning

#### Check Command
```bash
CLUSTER_NAME="<cluster-name>"
REGION="ap-northeast-1"

# Method 1: Check EKS Access Entries (modern)
aws eks list-access-entries --cluster-name "$CLUSTER_NAME" --region "$REGION" --output json 2>/dev/null

# Method 2: Check aws-auth ConfigMap (traditional)
kubectl get configmap aws-auth -n kube-system -o json 2>/dev/null | jq '{mapRoles: .data.mapRoles, mapUsers: .data.mapUsers}'
```

#### PASS Criteria
Either access entries exist (non-empty `accessEntries` list) OR `aws-auth` ConfigMap has `mapRoles`/`mapUsers` configured.

---

### C3: Running Large Clusters
**Severity**: 🟢 Info

#### Check Command
```bash
# Count total services
SERVICE_COUNT=$(kubectl get services --all-namespaces --no-headers 2>/dev/null | wc -l)
echo "Total services: $SERVICE_COUNT"

# If > 1000, check kube-proxy mode
kubectl get configmap kube-proxy-config -n kube-system -o json 2>/dev/null | jq '.data."config"' | grep -o '"mode":"[^"]*"'

# Check VPC CNI WARM_IP_TARGET
kubectl get daemonset aws-node -n kube-system -o json 2>/dev/null | jq '[.spec.template.spec.containers[0].env[] | select(.name | test("WARM_IP_TARGET|WARM_ENI_TARGET|MINIMUM_IP_TARGET"))]'
```

#### PASS Criteria
- If service count < 1000: automatic PASS (no optimization needed)
- If service count >= 1000: kube-proxy mode is `"ipvs"` AND `WARM_IP_TARGET` is set

---

### C4: EKS Control Plane Endpoint Access Control
**Severity**: 🔴 Critical

#### Check Command
```bash
CLUSTER_NAME="<cluster-name>"
REGION="ap-northeast-1"

aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" --query 'cluster.resourcesVpcConfig.{endpointPublicAccess: endpointPublicAccess, endpointPrivateAccess: endpointPrivateAccess, publicAccessCidrs: publicAccessCidrs}' --output json
```

#### PASS Criteria
One of:
- `endpointPublicAccess: false` (fully private) — PASS
- `endpointPublicAccess: true` AND `publicAccessCidrs` does NOT contain `"0.0.0.0/0"` — PASS
- `endpointPublicAccess: true` AND `publicAccessCidrs` contains `"0.0.0.0/0"` — FAIL

---

### C5: Avoid Catch-All Admission Webhooks
**Severity**: 🟡 Warning

#### Check Command
```bash
# Check MutatingWebhookConfigurations
kubectl get mutatingwebhookconfigurations -o json | jq '[.items[] | {name: .metadata.name, webhooks: [.webhooks[] | select((.namespaceSelector == null) and (.objectSelector == null) and (.rules[]? | (.apiGroups[]? == "*") or (.apiVersions[]? == "*") or (.resources[]? == "*"))) | {name: .name, rules: .rules}]} | select(.webhooks | length > 0)]'

# Check ValidatingWebhookConfigurations
kubectl get validatingwebhookconfigurations -o json | jq '[.items[] | {name: .metadata.name, webhooks: [.webhooks[] | select((.namespaceSelector == null) and (.objectSelector == null) and (.rules[]? | (.apiGroups[]? == "*") or (.apiVersions[]? == "*") or (.resources[]? == "*"))) | {name: .name, rules: .rules}]} | select(.webhooks | length > 0)]'
```

#### PASS Criteria
Both commands return `[]` — no webhooks with overly broad wildcard rules and missing selectors.

---

## Data Plane Checks (D1–D7)

### D1: Use Kubernetes Cluster Autoscaler or Karpenter
**Severity**: 🔴 Critical

#### Check Command
```bash
# Check for Cluster Autoscaler
kubectl get deployment -n kube-system -l app=cluster-autoscaler -o json 2>/dev/null | jq '.items | length'

# Check for Karpenter
kubectl get namespace karpenter 2>/dev/null && echo "Karpenter namespace found"
kubectl get deployment -n karpenter -o json 2>/dev/null | jq '[.items[] | {name: .metadata.name, available: .status.availableReplicas}]'
kubectl get crd nodepools.karpenter.sh 2>/dev/null && echo "Karpenter CRDs found"
kubectl get crd ec2nodeclasses.karpenter.k8s.aws 2>/dev/null && echo "Karpenter AWS CRDs found"
```

#### PASS Criteria
Either Cluster Autoscaler deployment exists with `availableReplicas >= 1`, OR Karpenter namespace/CRDs/deployments exist.

---

### D2: Worker Nodes Spread Across Multiple AZs
**Severity**: 🔴 Critical

#### Check Command
```bash
# List nodes with AZ labels
kubectl get nodes -o json | jq '[.items[] | {name: .metadata.name, az: .metadata.labels["topology.kubernetes.io/zone"]}] | group_by(.az) | map({az: .[0].az, count: length})'

# Check AZ balance (variance within 20%)
kubectl get nodes -o json | jq '[.items[] | .metadata.labels["topology.kubernetes.io/zone"]] | group_by(.) | map({az: .[0], count: length}) | (map(.count) | (max - min) / max * 100) as $variance | {az_distribution: ., variance_percent: $variance, balanced: ($variance <= 20)}'
```

#### PASS Criteria
- Nodes are spread across 2+ AZs
- Distribution variance is ≤ 20% (`balanced: true`)

---

### D3: Configure Resource Requests/Limits
**Severity**: 🔴 Critical

#### Check Command
```bash
kubectl get deployments --all-namespaces -o json | jq '[.items[] | select(.metadata.namespace | test("^kube-") | not) | {namespace: .metadata.namespace, name: .metadata.name, containers_missing_resources: [.spec.template.spec.containers[] | {name: .name, has_cpu_request: (.resources.requests.cpu != null), has_cpu_limit: (.resources.limits.cpu != null), has_mem_request: (.resources.requests.memory != null), has_mem_limit: (.resources.limits.memory != null)} | select(.has_cpu_request == false or .has_cpu_limit == false or .has_mem_request == false or .has_mem_limit == false)]} | select(.containers_missing_resources | length > 0)]'
```

#### PASS Criteria
Empty array `[]` — all containers in all Deployments have CPU and memory requests AND limits.

---

### D4: Namespace ResourceQuotas
**Severity**: 🟡 Warning

#### Check Command
```bash
# List user namespaces without ResourceQuota
kubectl get namespaces -o json | jq '[.items[] | select(.metadata.name | test("^kube-|^amazon-|^istio-|^linkerd") | not) | .metadata.name]' | while read -r ns; do
  ns=$(echo "$ns" | tr -d '"[] ')
  [ -z "$ns" ] && continue
  count=$(kubectl get resourcequota -n "$ns" --no-headers 2>/dev/null | wc -l)
  if [ "$count" -eq 0 ]; then echo "MISSING ResourceQuota: $ns"; fi
done

# Alternative single-command approach
for ns in $(kubectl get namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep -v -E '^kube-|^amazon-'); do
  if [ "$(kubectl get resourcequota -n "$ns" --no-headers 2>/dev/null | wc -l)" -eq 0 ]; then
    echo "MISSING: $ns"
  fi
done
```

#### PASS Criteria
No output — all user namespaces (including `default`) have at least one ResourceQuota.

---

### D5: Namespace LimitRanges
**Severity**: 🟡 Warning

#### Check Command
```bash
for ns in $(kubectl get namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep -v -E '^kube-|^amazon-'); do
  if [ "$(kubectl get limitrange -n "$ns" --no-headers 2>/dev/null | wc -l)" -eq 0 ]; then
    echo "MISSING: $ns"
  fi
done
```

#### PASS Criteria
No output — all user namespaces have at least one LimitRange.

---

### D6: Monitor CoreDNS Metrics
**Severity**: 🟡 Warning

#### Check Command
```bash
# Verify CoreDNS has metrics port 9153
kubectl get deployment coredns -n kube-system -o json 2>/dev/null | jq '[.spec.template.spec.containers[0].ports[] | select(.containerPort == 9153)]'

# Check for ServiceMonitor targeting CoreDNS
kubectl get servicemonitor -n kube-system -o json 2>/dev/null | jq '[.items[] | select(.spec.selector.matchLabels["k8s-app"] == "kube-dns" or .metadata.name | test("coredns|dns")) | {name: .metadata.name}]'

# Check for Prometheus scrape annotations on CoreDNS service
kubectl get service kube-dns -n kube-system -o json 2>/dev/null | jq '.metadata.annotations | with_entries(select(.key | test("prometheus")))'
```

#### PASS Criteria
CoreDNS metrics port 9153 exists AND (ServiceMonitor targeting CoreDNS exists OR Prometheus scrape annotations present).

---

### D7: CoreDNS Configuration
**Severity**: 🟢 Info

#### Check Command
```bash
CLUSTER_NAME="<cluster-name>"
REGION="ap-northeast-1"

# Check if EKS Auto Mode
aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" --query 'cluster.computeConfig.enabled' --output text 2>/dev/null

# Check if CoreDNS is an EKS managed add-on
aws eks describe-addon --cluster-name "$CLUSTER_NAME" --addon-name coredns --region "$REGION" --query 'addon.{status: status, version: addonVersion, configurationSchema: configurationSchema}' --output json 2>/dev/null

# Verify CoreDNS deployment exists
kubectl get deployment coredns -n kube-system -o json 2>/dev/null | jq '{name: .metadata.name, replicas: .spec.replicas, available: .status.availableReplicas}'
```

#### PASS Criteria
- EKS Auto Mode enabled → automatic PASS
- OR CoreDNS is an EKS managed add-on (describe-addon returns status)
- FAIL if CoreDNS is self-managed in non-auto-mode clusters
