# FAIL 到实验映射

> 将 EKS 韧性检查失败项映射到推荐的混沌实验。
> 每个映射包含原理、假设、持续时间、预期影响和成功标准。

---

## 概述

| Check FAIL | Fault Type | Backend | Priority | Hypothesis |
|------------|-----------|---------|----------|------------|
| A1: Singleton Pod | pod_kill | chaosmesh | P0 | Killing singleton pod causes permanent service loss until manual intervention |
| A2: Single Replica | pod_kill / fis_eks_pod_delete | chaosmesh/fis | P0 | Single-replica service has guaranteed downtime (~30-60s) on pod failure |
| A3: No Anti-Affinity | fis_eks_terminate_node | fis | P1 | All replicas on same node = single point of failure |
| A4: No Liveness Probe | pod_cpu_stress | chaosmesh | P1 | Zombie process won't be detected/restarted without probe |
| A5: No Readiness Probe | network_delay | chaosmesh | P1 | Traffic still routes to degraded pod without readiness gate |
| A6: No PDB | fis_eks_terminate_node | fis | P1 | Node drain evicts all replicas simultaneously |
| A8: No HPA | pod_cpu_stress | chaosmesh | P2 | Cannot auto-scale under load spike |
| D1: No Node Autoscaler | fis_ssm_cpu_stress | fis | P1 | Node resource exhaustion blocks new pod scheduling |
| D2: Single AZ | fis_network_disrupt / fis_scenario_az_power_interruption | fis | P0 | AZ failure = total cluster outage |
| D3: No Resource Limits | pod_memory_stress | chaosmesh | P1 | Noisy neighbor affects co-located pods |

---

## 详细映射

### A1: Singleton Pod → pod_kill

**优先级**: P0

**原理**:
Singleton pods (pods without a controller like Deployment or ReplicaSet) have no automatic recovery mechanism. When a singleton pod is killed, Kubernetes does not recreate it — the service remains down until an operator manually intervenes. This is the most basic resilience gap and represents a guaranteed single point of failure. A chaos experiment validates that this gap is real and quantifies the blast radius.

**假设**:
> **Steady State**: The application service backed by the singleton pod returns HTTP 200 with p99 latency < 500ms.
> **Turbulent Action**: Kill the singleton pod using `pod_kill` fault injection.
> **Expected Outcome**: The service becomes completely unavailable. No automatic recovery occurs within the observation window. Service remains down until manual pod recreation.

**建议持续时间**: 3 minutes observation after pod kill

**预期影响**:
- Complete service unavailability for the affected endpoint
- Dependent services may cascade-fail if they have no timeout/retry logic
- No automatic recovery — pod stays terminated

**成功标准**（修复后的韧性表现）:
- Pod is managed by a Deployment with `replicas >= 2`
- After pod kill, remaining replica(s) continue serving traffic
- Killed pod is automatically recreated by the controller within 30s
- Service availability remains > 99% during the experiment

---

### A2: Single Replica → pod_kill / fis_eks_pod_delete

**优先级**: P0

**原理**:
A Deployment or StatefulSet with `replicas: 1` has a controller that will recreate the pod, but there is guaranteed downtime between termination and the new pod becoming ready. This gap — typically 30-60 seconds depending on image pull time, startup time, and readiness probe configuration — is real production impact. For customer-facing services, this means dropped requests and broken user sessions. This experiment measures the exact recovery time to inform SLA commitments.

**假设**:
> **Steady State**: The service handles requests at normal throughput with p99 latency < 500ms and zero 5xx errors.
> **Turbulent Action**: Delete the single pod via `pod_kill` (ChaosMesh) or `fis_eks_pod_delete` (FIS).
> **Expected Outcome**: Service is unavailable for 30-60 seconds. The controller recreates the pod. Recovery time depends on image pull + startup + readiness probe.

**建议持续时间**: 5 minutes (including recovery observation)

**预期影响**:
- 100% traffic loss for the target service during pod recreation
- Upstream services receive connection refused / 503 errors
- Recovery is automatic but not instantaneous
- Possible data loss for in-flight transactions without idempotency

**成功标准**（修复后的韧性表现）:
- Deployment scaled to `replicas >= 2`
- Pod kill removes one replica; remaining replica(s) handle 100% of traffic
- No 5xx errors observed at the load balancer during the experiment
- Failed pod is recreated and passes readiness within 30s

---

### A3: No Anti-Affinity → fis_eks_terminate_node

**优先级**: P1

**原理**:
Without pod anti-affinity rules, the Kubernetes scheduler may place all replicas of a Deployment on the same node. While you have multiple replicas on paper, a single node failure takes them all down simultaneously — the high-availability benefit of multi-replica is negated. This experiment terminates a node to validate whether replicas are actually distributed. If all replicas are co-located, the result is equivalent to a single-replica failure.

**假设**:
> **Steady State**: The multi-replica service processes requests across all pods with balanced load distribution.
> **Turbulent Action**: Terminate the EC2 instance hosting the application pods using `fis_eks_terminate_node`.
> **Expected Outcome**: If anti-affinity is missing and all pods are on the terminated node, service experiences complete outage until pods are rescheduled to surviving nodes. If pods happen to be spread, service continues but this is luck-dependent, not guaranteed.

**建议持续时间**: 10 minutes (node termination + pod rescheduling + stabilization)

**预期影响**:
- If all replicas co-located: complete service outage lasting 2-5 minutes (pod rescheduling time)
- If replicas happen to be distributed: partial impact only
- Node-level workloads (DaemonSets) are temporarily lost on the terminated node
- Auto Scaling Group or Karpenter recreates the node

**成功标准**（修复后的韧性表现）:
- `podAntiAffinity` configured with `topologyKey: kubernetes.io/hostname`
- Replicas verified running on different nodes before experiment
- Node termination causes loss of at most one replica
- Remaining replicas absorb traffic with < 10% latency increase
- PDB prevents simultaneous eviction during node drain

---

### A4: No Liveness Probe → pod_cpu_stress

**优先级**: P1

**原理**:
Without a liveness probe, Kubernetes has no mechanism to detect that a container has become unhealthy (deadlocked, stuck in infinite loop, or unresponsive). The container continues running and consuming resources while providing no useful work — a "zombie process." CPU stress simulates a scenario where the application becomes unresponsive under load. With a liveness probe, Kubernetes would restart the container; without one, the degraded state persists indefinitely.

**假设**:
> **Steady State**: Application serves requests with p99 latency < 500ms and CPU usage < 50%.
> **Turbulent Action**: Inject CPU stress into the target pod using `pod_cpu_stress` (90% CPU for 2 minutes).
> **Expected Outcome**: The pod becomes unresponsive. Without a liveness probe, Kubernetes does NOT restart the container. The pod stays in `Running` state but cannot serve traffic. Service degradation persists until manual intervention.

**建议持续时间**: 5 minutes (2 min stress + 3 min observation)

**预期影响**:
- Affected pod stops responding to requests but remains in `Running` state
- If readiness probe exists: pod is removed from endpoints (partial mitigation)
- If no readiness probe either: traffic continues routing to the zombie pod, causing errors
- No automatic recovery — container is never restarted by kubelet

**成功标准**（修复后的韧性表现）:
- Liveness probe configured (e.g., HTTP health endpoint)
- Under CPU stress, liveness probe fails after `failureThreshold` attempts
- Kubelet restarts the container automatically within 30-45s
- Restarted container recovers normal operation
- Service availability maintained by other healthy replicas during restart

---

### A5: No Readiness Probe → network_delay

**优先级**: P1

**原理**:
Without a readiness probe, Kubernetes considers a pod "ready" to receive traffic as soon as its containers are running — even if the application is still initializing, or has become degraded due to network issues. Network delay injection simulates a scenario where the pod's backend connectivity degrades, making it unable to serve requests properly. Without readiness probes, the Service continues sending traffic to this degraded pod, resulting in slow responses or errors for end users.

**假设**:
> **Steady State**: Application handles requests with p99 latency < 500ms and error rate < 0.1%.
> **Turbulent Action**: Inject 3-second network delay on the target pod's egress using `network_delay`.
> **Expected Outcome**: The pod's responses become extremely slow (3s+ added latency). Without a readiness probe, the Kubernetes Service continues routing traffic to this degraded pod. Users experience timeouts and errors proportional to the percentage of traffic hitting the affected pod.

**建议持续时间**: 5 minutes

**预期影响**:
- Fraction of requests (1/N where N = replica count) experience 3s+ additional latency
- Requests may timeout at the load balancer or upstream service level
- Error rate increases proportionally to traffic routed to the affected pod
- No automatic traffic diversion — Service keeps the degraded pod in its endpoints

**成功标准**（修复后的韧性表现）:
- Readiness probe configured to check actual serving capability
- Under network delay, readiness probe fails within 10-15s
- Pod is automatically removed from Service endpoints
- Remaining healthy pods absorb 100% of traffic
- Error rate stays < 0.1% during the experiment
- When delay is removed, pod passes readiness and is re-added to endpoints

---

### A6: No PDB → fis_eks_terminate_node

**优先级**: P1

**原理**:
Without a PodDisruptionBudget (PDB), voluntary disruptions like node drains (during cluster upgrades, maintenance, or scaling) can evict all replicas of a workload simultaneously. The Kubernetes API server has no constraint to preserve availability — if 3 replicas are on the same node being drained, all 3 are evicted at once. This turns a controlled, planned operation into an unplanned outage. The experiment validates that a node termination (which triggers a drain) does not violate the application's availability requirements.

**假设**:
> **Steady State**: The multi-replica service serves traffic with zero downtime. All replicas are healthy.
> **Turbulent Action**: Terminate a worker node using `fis_eks_terminate_node`, which triggers pod eviction.
> **Expected Outcome**: Without PDB, the drain process evicts pods as fast as possible with no availability guarantee. If multiple replicas are on the same node, all are evicted simultaneously, causing a service outage until rescheduling completes.

**建议持续时间**: 10 minutes (drain + reschedule + stabilization)

**预期影响**:
- All replicas on the terminated node are evicted simultaneously
- Brief or extended service outage depending on replica distribution
- No controlled rollout of evictions — Kubernetes drains as fast as possible
- Rescheduling to other nodes adds 30-120s of unavailability

**成功标准**（修复后的韧性表现）:
- PDB configured with `minAvailable: 1` or `maxUnavailable: 1`
- Node drain respects PDB — evicts pods one at a time
- At least one replica remains serving traffic throughout the drain
- Service experiences zero downtime during the experiment
- All pods rescheduled successfully on surviving nodes

---

### A8: No HPA → pod_cpu_stress

**优先级**: P2

**原理**:
Without a Horizontal Pod Autoscaler (HPA), the application cannot automatically scale out when demand increases. If existing replicas are overwhelmed by traffic, there is no mechanism to add capacity — latency increases, requests queue up, and eventually the service degrades or fails. CPU stress simulates a load spike. With HPA, the cluster would automatically add replicas when CPU utilization exceeds the threshold; without HPA, the application runs at degraded performance until manual intervention.

**假设**:
> **Steady State**: Application serves requests with p99 latency < 500ms at normal load. Replica count is stable.
> **Turbulent Action**: Inject CPU stress (80% CPU) on all existing pods simultaneously using `pod_cpu_stress`.
> **Expected Outcome**: All pods become CPU-saturated. Without HPA, no additional replicas are created. Latency increases significantly (5-10x). Throughput drops. Service remains in degraded state for the duration of the stress.

**建议持续时间**: 5 minutes stress + 5 minutes recovery observation

**预期影响**:
- Application latency increases 5-10x under CPU saturation
- Throughput drops as pods cannot process requests fast enough
- No automatic scaling occurs — replica count stays static
- Possible cascading failures in upstream services due to timeouts

**成功标准**（修复后的韧性表现）:
- HPA configured with CPU target utilization (e.g., 70%)
- Under CPU stress, HPA triggers scale-out within 30-60s
- New replicas start and pass readiness checks
- Traffic is distributed across original + new replicas
- Latency returns to acceptable levels (< 1s p99) within 2 minutes of scale-out
- After stress removal, HPA scales down gracefully (stabilization window)

---

### D1: No Node Autoscaler → fis_ssm_cpu_stress

**优先级**: P1

**原理**:
Without a node autoscaler (Cluster Autoscaler or Karpenter), the cluster has a fixed compute capacity ceiling. When all nodes are fully utilized, new pods cannot be scheduled — they remain in `Pending` state indefinitely. This blocks both HPA scale-out and any new deployments. CPU stress on all nodes simulates resource exhaustion, validating whether the cluster can grow to accommodate increased demand. This is a data-plane-level resilience gap that affects all workloads.

**假设**:
> **Steady State**: Cluster has sufficient capacity. All pods are `Running`. No `Pending` pods exist.
> **Turbulent Action**: Inject CPU stress on all worker nodes using `fis_ssm_cpu_stress` (90% CPU), then trigger a workload that requires new pod scheduling.
> **Expected Outcome**: Existing nodes are fully utilized. New pods enter `Pending` state. Without node autoscaler, no new EC2 instances are launched. Pods remain `Pending` indefinitely until stress is removed or manual node scaling occurs.

**建议持续时间**: 10 minutes (stress + scheduling attempt + observation)

**预期影响**:
- All existing pods experience degraded performance due to CPU contention
- New pods cannot be scheduled — remain `Pending`
- HPA scale-out is blocked even if triggered (no available capacity)
- No automatic capacity expansion
- Cluster resource metrics show 100% utilization

**成功标准**（修复后的韧性表现）:
- Karpenter or Cluster Autoscaler is deployed and operational
- Under node resource pressure, autoscaler detects `Pending` pods within 30s
- New EC2 instances are launched within 2-3 minutes
- `Pending` pods are scheduled on new nodes
- Cluster capacity adjusts dynamically to meet demand
- After stress removal, underutilized nodes are reclaimed (scale-down)

---

### D2: Single AZ → fis_network_disrupt / fis_scenario_az_power_interruption

**优先级**: P0

**原理**:
If all worker nodes are in a single Availability Zone, an AZ-level failure (network partition, power loss, or infrastructure issue) takes down the entire cluster's compute capacity. This is the highest-impact data plane risk — no amount of application-level resilience (multiple replicas, probes, PDBs) can compensate for losing all underlying infrastructure. AZ failures are rare but real (AWS has documented incidents). This experiment simulates an AZ disruption to validate that the cluster can survive the loss of an entire AZ.

**假设**:
> **Steady State**: Application serves traffic normally. All nodes and pods are healthy across available AZs.
> **Turbulent Action**: Disrupt network connectivity to the AZ hosting all nodes using `fis_network_disrupt`, or simulate a complete AZ power interruption using `fis_scenario_az_power_interruption`.
> **Expected Outcome**: All nodes in the affected AZ become `NotReady`. All pods on those nodes are evicted after the node `NotReady` toleration timeout (default 5 minutes). If single-AZ, the entire cluster workload is lost. No failover is possible because there are no nodes in other AZs.

**建议持续时间**: 15 minutes (disruption + eviction timeout + recovery observation)

**预期影响**:
- **Single AZ cluster**: Total outage. All pods evicted. No failover possible.
- All services become unavailable
- No automatic recovery until AZ connectivity is restored or new nodes are launched in other AZs
- Data on local storage (emptyDir, hostPath) is lost
- PersistentVolumes bound to the affected AZ are inaccessible

**成功标准**（修复后的韧性表现）:
- Nodes distributed across 3 AZs with balanced distribution (± 20% variance)
- Pod anti-affinity ensures replicas span AZs (via `topology.kubernetes.io/zone`)
- AZ disruption causes loss of ~1/3 of pods
- Remaining pods in surviving AZs continue serving traffic
- Load balancer health checks route traffic away from the affected AZ
- Service availability remains > 95% during the AZ outage
- Node autoscaler replaces lost capacity in surviving AZs

---

### D3: No Resource Limits → pod_memory_stress

**优先级**: P1

**原理**:
Without resource limits, a single container can consume unbounded CPU and memory on its host node. This creates a "noisy neighbor" problem — one misbehaving pod starves co-located pods of resources, degrading unrelated services. Memory stress is particularly dangerous because OOM conditions can kill other pods on the node, not just the stressed one. This experiment validates whether a memory-hungry pod can impact other workloads on the same node.

**假设**:
> **Steady State**: Multiple pods on the same node each serve their traffic normally. Memory usage is within expected ranges.
> **Turbulent Action**: Inject memory stress into one pod (consuming 1-2 GB additional memory) using `pod_memory_stress`.
> **Expected Outcome**: The stressed pod's memory usage grows without constraint. Node memory pressure increases. The kubelet begins evicting pods based on QoS class. Co-located pods (especially BestEffort class, which have no resource specs) are evicted first. Services running on the same node experience disruption unrelated to their own behavior.

**建议持续时间**: 5 minutes

**预期影响**:
- Stressed pod consumes memory beyond its fair share
- Node enters `MemoryPressure` condition
- Kubelet evicts BestEffort and Burstable pods to recover memory
- Co-located services experience unexpected restarts / evictions
- If OOM killer is triggered, it may kill any container on the node
- Cascading impact on services that were functioning normally

**成功标准**（修复后的韧性表现）:
- All containers have memory requests and limits set
- Memory stress hits the pod's limit → container is OOM-killed (contained blast radius)
- Kubernetes restarts only the offending container (via `restartPolicy: Always`)
- Co-located pods are NOT evicted or affected
- Node does not enter `MemoryPressure` condition
- Service impact is limited to the single stressed pod's restart time
