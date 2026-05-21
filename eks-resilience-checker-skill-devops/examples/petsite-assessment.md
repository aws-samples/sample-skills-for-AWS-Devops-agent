# EKS Resilience Assessment Report — PetSite

**Cluster**: PetSite  
**Region**: ap-northeast-1  
**Kubernetes Version**: 1.32  
**Platform Version**: eks.8  
**Assessment Date**: 2026-04-03T08:00:00Z  
**Target Namespaces**: petadoptions, default

---

## Executive Summary

| Metric | Value |
|--------|-------|
| Total Checks | 28 |
| Passed | 20 |
| Failed | 6 |
| Info | 2 |
| Critical Failures | 3 |
| **Compliance Score** | **71.4%** |

```
Score Breakdown:
  Application (A1-A14):   9/14 PASS   (64.3%)
  Control Plane (C1-C5):  4/5  PASS   (80.0%)
  Data Plane (D1-D7):     7/7  PASS*  (100.0%)
  * D4, D5 are FAIL but categorized under Application score adjustment below

Overall: 20/28 PASS = 71.4%
```

### Risk Distribution

```
Critical:  ██████░░░░  3 FAIL / 7 total critical checks
Warning:   ███░░░░░░░  3 FAIL / 11 total warning checks
Info:      ██████████  0 FAIL / 4 total info checks (2 INFO status)
```

---

## Detailed Findings

### Application Checks (A1-A14)

#### A1: Avoid Singleton Pods — PASS

No standalone pods found. All pods are managed by controllers (Deployment, StatefulSet, DaemonSet).

#### A2: Run Multiple Replicas — FAIL (Critical)

| Deployment | Namespace | Replicas | Status |
|-----------|-----------|----------|--------|
| petlistadoptions | petadoptions | 2 | OK |
| petsearch | petadoptions | 2 | OK |
| petsite | petadoptions | 2 | OK |
| payforadoption | petadoptions | 1 | **FAIL** |
| pethistory | petadoptions | 1 | **FAIL** |
| statusupdater | petadoptions | 1 | **FAIL** |

**Finding**: 3 Deployments running with single replicas — guaranteed downtime on pod failure.

**Remediation**:
```bash
kubectl scale deployment payforadoption --replicas=2 -n petadoptions
kubectl scale deployment pethistory --replicas=2 -n petadoptions
kubectl scale deployment statusupdater --replicas=2 -n petadoptions
```

#### A3: Use Pod Anti-Affinity — FAIL (Warning)

| Deployment | Replicas | Anti-Affinity | Status |
|-----------|----------|---------------|--------|
| petlistadoptions | 2 | Not configured | **FAIL** |
| petsearch | 2 | Not configured | **FAIL** |
| petsite | 2 | Not configured | **FAIL** |

**Finding**: All multi-replica Deployments lack podAntiAffinity — all replicas could be scheduled on the same node.

**Remediation**:
```yaml
# Add to each Deployment spec.template.spec
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - <app-name>
        topologyKey: kubernetes.io/hostname
```

#### A4: Use Liveness Probes — PASS

All containers across Deployments, StatefulSets, and DaemonSets have livenessProbe configured.

#### A5: Use Readiness Probes — PASS

All containers have readinessProbe configured.

#### A6: Use Pod Disruption Budgets — FAIL (Warning)

| Workload | Type | Replicas | PDB | Status |
|----------|------|----------|-----|--------|
| petlistadoptions | Deployment | 2 | None | **FAIL** |
| petsearch | Deployment | 2 | None | **FAIL** |
| petsite | Deployment | 2 | None | **FAIL** |

**Finding**: No PodDisruptionBudgets found for multi-replica workloads. Node drain during maintenance could terminate all replicas simultaneously.

**Remediation**:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: petlistadoptions-pdb
  namespace: petadoptions
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: petlistadoptions
EOF
# Repeat for petsearch, petsite
```

#### A7: Run Kubernetes Metrics Server — PASS

metrics-server running in kube-system. `kubectl top nodes` responds successfully.

#### A8: Use Horizontal Pod Autoscaler — FAIL (Warning)

| Workload | Replicas | HPA | Status |
|----------|----------|-----|--------|
| petlistadoptions | 2 | None | **FAIL** |
| petsearch | 2 | None | **FAIL** |
| petsite | 2 | None | **FAIL** |

**Finding**: Multi-replica workloads have no HPA — cannot auto-scale under load.

**Remediation**:
```bash
kubectl autoscale deployment petlistadoptions --cpu-percent=70 --min=2 --max=5 -n petadoptions
kubectl autoscale deployment petsearch --cpu-percent=70 --min=2 --max=5 -n petadoptions
kubectl autoscale deployment petsite --cpu-percent=70 --min=2 --max=5 -n petadoptions
```

#### A9: Use Custom Metrics Scaling — INFO

No custom metrics infrastructure (KEDA, Prometheus Adapter) detected. Basic CPU/memory scaling only.

> This is informational — custom metrics are recommended but not required for all clusters.

#### A10: Use Vertical Pod Autoscaler — INFO

VPA CRD and controller not installed. Workloads rely on manually configured resource requests/limits.

> This is informational — VPA is recommended for right-sizing but not critical.

#### A11: Use PreStop Hooks — PASS

All Deployment and StatefulSet containers have `lifecycle.preStop` hooks configured for graceful termination.

#### A12: Use a Service Mesh — PASS

AWS App Mesh detected (appmesh-controller running in appmesh-system namespace). Sidecar proxies injected in petadoptions namespace pods.

#### A13: Monitor Your Applications — PASS

CloudWatch Container Insights enabled. ADOT Collector DaemonSet running for metrics and traces collection.

#### A14: Use Centralized Logging — PASS

Fluent Bit DaemonSet running in amazon-cloudwatch namespace. Logs shipping to CloudWatch Logs.

---

### Control Plane Checks (C1-C5)

#### C1: Monitor Control Plane Logs — PASS

Control plane logging enabled:
- api: Enabled
- audit: Enabled
- authenticator: Enabled
- controllerManager: Disabled
- scheduler: Disabled

> Recommendation: Consider enabling controllerManager and scheduler logs for full visibility.

#### C2: Cluster Authentication — PASS

EKS Access Entries configured (modern API-based method). 3 access entries found with appropriate policies.

#### C3: Running Large Clusters — PASS

Total services: 42 (well below 1,000 threshold). No large-cluster optimizations needed.

#### C4: API Server Endpoint Access Control — FAIL (Critical)

| Setting | Value | Status |
|---------|-------|--------|
| Public Access | Enabled | - |
| Private Access | Enabled | - |
| Public CIDR | 0.0.0.0/0 | **FAIL** |

**Finding**: API server publicly accessible from any IP address. This is a critical security risk.

**Remediation**:
```bash
aws eks update-cluster-config \
  --name PetSite \
  --region ap-northeast-1 \
  --resources-vpc-config \
    endpointPublicAccess=true,\
    publicAccessCidrs="203.0.113.0/24",\
    endpointPrivateAccess=true
```

#### C5: Avoid Catch-All Admission Webhooks — PASS

No overly broad MutatingWebhook or ValidatingWebhook configurations detected. All webhooks have proper namespace and object selectors.

---

### Data Plane Checks (D1-D7)

#### D1: Use Cluster Autoscaler or Karpenter — PASS

Karpenter v0.37 detected (karpenter namespace). NodePool and EC2NodeClass resources configured.

#### D2: Worker Nodes Spread Across Multiple AZs — PASS

| Availability Zone | Node Count | Percentage |
|------------------|------------|------------|
| ap-northeast-1a | 3 | 33.3% |
| ap-northeast-1c | 3 | 33.3% |
| ap-northeast-1d | 3 | 33.3% |

Distribution variance: 0% (within 20% threshold). Excellent AZ balance.

#### D3: Configure Resource Requests/Limits — PASS

All Deployments in petadoptions namespace have complete CPU and memory requests and limits.

#### D4: Namespace ResourceQuotas — PASS

| Namespace | ResourceQuota | Status |
|-----------|--------------|--------|
| petadoptions | petadoptions-quota | OK |
| default | default-quota | OK |

#### D5: Namespace LimitRanges — PASS

| Namespace | LimitRange | Status |
|-----------|-----------|--------|
| petadoptions | petadoptions-limits | OK |
| default | default-limits | OK |

#### D6: Monitor CoreDNS Metrics — PASS

CoreDNS running with metrics port 9153. ServiceMonitor configured for Prometheus scraping.

#### D7: CoreDNS Managed Configuration — PASS

CoreDNS managed via EKS Managed Add-on (coredns v1.11.4-eksbuild.2). Automatic updates enabled.

---

## Experiment Recommendations

Based on FAIL findings, the following chaos experiments are recommended:

| Priority | Check | Target | Experiment | Hypothesis |
|----------|-------|--------|------------|------------|
| **P0** | A2 | payforadoption (replicas=1) | Pod kill | Killing the single-replica pod will cause service unavailability for ~30-60s until K8s recreates the pod |
| **P0** | A2 | pethistory (replicas=1) | Pod kill | Single-replica pod failure causes complete feature unavailability |
| **P0** | C4 | API server endpoint | Network scan | Unrestricted public access exposes cluster to unauthorized API calls |
| **P1** | A3 | petlistadoptions, petsearch, petsite | Node termination | Without anti-affinity, all replicas may co-locate — node failure could take down entire service |
| **P1** | A6 | petlistadoptions, petsearch, petsite | Node drain | Without PDB, node drain during maintenance could evict all replicas simultaneously |
| **P2** | A8 | petlistadoptions, petsearch, petsite | CPU stress | Without HPA, traffic spikes cannot trigger automatic scaling — verify degradation under load |

### Recommended Experiment Sequence

```
Phase 1 (P0 — Immediate):
  1. Pod kill on payforadoption → measure actual downtime
  2. Pod kill on pethistory → measure actual downtime
  3. Restrict API server access (fix, not experiment)

Phase 2 (P1 — After fixing replicas):
  4. Terminate node hosting petlistadoptions pods → verify anti-affinity gap
  5. kubectl drain on a worker node → verify PDB gap

Phase 3 (P2 — After adding PDB + anti-affinity):
  6. CPU stress on petsearch → verify no auto-scaling response
```

### Integration with chaos-engineering-on-aws

To run these experiments using the chaos skill:

```
1. Ensure chaos-engineering-on-aws skill is installed
2. Tell your AI agent:
   "Run chaos experiments based on the EKS assessment at output/assessment.json"
3. The chaos skill will read experiment_recommendations and guide you through
   the 6-step experiment lifecycle
```

---

## Remediation Summary

### Critical (Fix Immediately)

| # | Issue | Command |
|---|-------|---------|
| 1 | payforadoption single replica | `kubectl scale deployment payforadoption --replicas=2 -n petadoptions` |
| 2 | pethistory single replica | `kubectl scale deployment pethistory --replicas=2 -n petadoptions` |
| 3 | statusupdater single replica | `kubectl scale deployment statusupdater --replicas=2 -n petadoptions` |
| 4 | API server unrestricted access | `aws eks update-cluster-config --name PetSite --resources-vpc-config publicAccessCidrs="<your-cidr>"` |

### Warning (Fix Soon)

| # | Issue | Action |
|---|-------|--------|
| 5 | No Pod Anti-Affinity | Add podAntiAffinity to multi-replica Deployments |
| 6 | No PDB | Create PodDisruptionBudgets for critical workloads |
| 7 | No HPA | Create HorizontalPodAutoscalers for multi-replica workloads |

### Info (Consider)

| # | Item | Recommendation |
|---|------|---------------|
| 8 | No custom metrics | Consider KEDA or Prometheus Adapter for business metric scaling |
| 9 | No VPA | Consider VPA for automated resource right-sizing |

---

*Generated by eks-resilience-checker v1.0 | Full structured data: output/assessment.json*
