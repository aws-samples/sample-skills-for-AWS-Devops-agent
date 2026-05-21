# EKS 韧性检查项

> **范围**：本文件说明每项检查**做什么**以及**为什么重要**（描述、影响、背景）。
> 关于**如何执行**每项检查（命令 + PASS/FAIL 标准），请参见 [check-commands_zh.md](check-commands_zh.md)。

## 概述

EKS 韧性检查器是一个综合工具，根据 26 项关键韧性最佳实践评估 Amazon EKS 集群。它在三个主要类别中执行自动化检查：应用工作负载、控制平面配置和数据平面设置。该工具为每项检查提供详细发现、合规状态和可操作的修复指导。

## 检查类别

### 应用相关检查 (A1-A14)
这些检查专注于应用工作负载的韧性和运维最佳实践。

### 控制平面相关检查 (C1-C5)
这些检查评估 EKS 控制平面配置的安全性、监控和可扩展性。

### 数据平面相关检查 (D1-D7)
这些检查评估工作节点配置、资源管理和集群基础设施。

---

## 详细检查描述

### **A1: Avoid Running Singleton Pods**
**目的**: Identifies standalone pods that aren't managed by controllers (Deployments, StatefulSets, etc.)

**检查内容**:
- Scans all pods across specified namespaces
- Identifies pods without `ownerReferences` (indicating they're not managed by controllers)
- Excludes system namespaces from the check

**重要性**:
- Singleton pods have no automatic restart capability if they fail
- No rolling update or scaling capabilities
- Difficult to manage and maintain
- Single point of failure for applications

**合规判定**: ✅ PASS if no singleton pods found, ❌ FAIL if any exist

---

### **A2: Run Multiple Replicas**
**目的**: Ensures Deployments and StatefulSets have more than one replica for high availability

**检查内容**:
- Examines all Deployments and StatefulSets
- Identifies workloads with `replicas: 1`
- Provides list of single-replica workloads

**重要性**:
- Single replica = single point of failure
- No availability during updates or node failures
- Cannot handle traffic spikes
- Violates high availability principles

**合规判定**: ✅ PASS if all workloads have >1 replica, ❌ FAIL if any have only 1 replica

---

### **A3: Use Pod Anti-Affinity**
**目的**: Ensures multi-replica deployments spread pods across different nodes

**检查内容**:
- Focuses on Deployments with >1 replica
- Checks for `podAntiAffinity` configuration in pod specs
- Identifies deployments that could have pods scheduled on the same node

**重要性**:
- Prevents all replicas from running on the same node
- Protects against node-level failures
- Improves fault tolerance and availability
- Better resource distribution

**合规判定**: ✅ PASS if multi-replica deployments have anti-affinity, ❌ FAIL if missing

---

### **A4: Use Liveness Probes**
**目的**: Ensures containers have health checks to detect and restart unhealthy instances

**检查内容**:
- Scans Deployments, StatefulSets, and DaemonSets
- Verifies each container has a `livenessProbe` configured
- Checks across all specified namespaces

**重要性**:
- Automatically restarts unhealthy containers
- Prevents zombie processes and deadlocks
- Improves application reliability
- Essential for self-healing applications

**合规判定**: ✅ PASS if all containers have liveness probes, ❌ FAIL if any are missing

---

### **A5: Use Readiness Probes**
**目的**: Ensures containers signal when they're ready to receive traffic

**检查内容**:
- Examines all workload types (Deployments, StatefulSets, DaemonSets)
- Verifies each container has a `readinessProbe`
- Identifies containers that might receive traffic before being ready

**重要性**:
- Prevents traffic routing to unready pods
- Improves user experience during deployments
- Reduces failed requests and timeouts
- Essential for zero-downtime deployments

**合规判定**: ✅ PASS if all containers have readiness probes, ❌ FAIL if any are missing

---

### **A6: Use Pod Disruption Budgets**
**目的**: Protects critical workloads during voluntary disruptions (updates, scaling, maintenance)

**检查内容**:
- Identifies critical workloads (multi-replica Deployments, all StatefulSets)
- Checks for corresponding PodDisruptionBudget resources
- Matches PDB selectors with workload selectors

**重要性**:
- Prevents all replicas from being terminated simultaneously
- Maintains availability during cluster maintenance
- Controls the pace of rolling updates
- Essential for production workloads

**合规判定**: ✅ PASS if critical workloads have PDBs, ❌ FAIL if any are unprotected

---

### **A7: Run Kubernetes Metrics Server**
**目的**: Ensures the cluster has metrics collection capability for monitoring and autoscaling

**检查内容**:
- Looks for metrics-server deployment in kube-system namespace
- Tests accessibility of the metrics API endpoint
- Verifies metrics collection infrastructure is available

**重要性**:
- Required for Horizontal Pod Autoscaler (HPA)
- Enables `kubectl top` commands
- Foundation for monitoring and alerting
- Essential for resource-based scaling decisions

**合规判定**: ✅ PASS if metrics server is running, ❌ FAIL if not found or inaccessible

---

### **A8: Use Horizontal Pod Autoscaler**
**目的**: Identifies multi-replica workloads that could benefit from automatic scaling

**检查内容**:
- Finds Deployments and StatefulSets with >1 replica
- Excludes system namespaces
- Checks for existing HPA resources protecting these workloads
- Supports both autoscaling/v1 and autoscaling/v2 APIs

**重要性**:
- Automatically scales applications based on demand
- Improves resource utilization
- Handles traffic spikes without manual intervention
- Reduces costs by scaling down during low usage

**合规判定**: ✅ PASS if multi-replica workloads have HPAs, ❌ FAIL if unprotected workloads exist

---

### **A9: Use Custom Metrics Scaling**
**目的**: Checks for advanced scaling capabilities beyond basic CPU/memory metrics

**检查内容**:
- Verifies custom metrics API availability
- Looks for external metrics API
- Checks for Prometheus Adapter deployment
- Identifies KEDA (event-driven autoscaling) installation
- Finds HPAs using custom or external metrics

**重要性**:
- Enables scaling based on business metrics (queue length, response time)
- More sophisticated than basic resource metrics
- Better alignment with application performance
- Supports event-driven architectures

**合规判定**: ✅ PASS if custom metrics infrastructure exists, ❌ FAIL if only basic metrics available

---

### **A10: Use Vertical Pod Autoscaler**
**目的**: Ensures workloads have right-sizing capabilities for resource optimization

**检查内容**:
- Looks for VPA controller components (recommender, updater, admission controller)
- Verifies VPA CRD installation
- Checks for existing VPA resources
- Identifies deployments without VPA configuration
- Detects Goldilocks (VPA UI) if installed

**重要性**:
- Automatically adjusts resource requests/limits
- Prevents over/under-provisioning
- Improves cluster resource utilization
- Reduces costs through right-sizing

**合规判定**: Three scenarios evaluated:
- ✅ PASS if VPA is installed and used appropriately
- ❌ FAIL if VPA infrastructure missing
- ❌ FAIL if VPA installed but not used

---

### **A11: Use PreStop Hooks**
**目的**: Ensures applications handle termination gracefully (excludes DaemonSets)

**检查内容**:
- Examines Deployments and StatefulSets only
- Verifies containers have `lifecycle.preStop` hooks configured
- Intentionally excludes DaemonSets (system services don't need graceful termination)

**重要性**:
- Allows applications to finish processing requests
- Prevents data loss during pod termination
- Improves user experience during deployments
- Essential for stateful applications

**合规判定**: ✅ PASS if application workloads have preStop hooks, ❌ FAIL if missing

---

### **A12: Use a Service Mesh**
**目的**: Detects service mesh implementation for advanced networking and observability

**检查内容**:
- Looks for Istio components (namespaces, CRDs, deployments)
- Checks for Linkerd installation
- Identifies Consul service mesh
- Detects sidecar proxy containers in application pods

**重要性**:
- Provides traffic management and security
- Enables advanced observability and tracing
- Improves service-to-service communication
- Adds resilience patterns (circuit breaking, retries)

**合规判定**: ✅ PASS if any service mesh detected, ❌ FAIL if none found

---

### **A13: Monitor Your Applications**
**目的**: Ensures comprehensive monitoring solution is deployed

**检查内容**:
- Looks for Prometheus stack (deployments, CRDs, namespaces)
- Detects CloudWatch Container Insights
- Identifies third-party monitoring (Datadog, New Relic, Dynatrace)
- Checks for monitoring infrastructure components

**重要性**:
- Essential for observability and alerting
- Enables proactive issue detection
- Supports troubleshooting and performance optimization
- Required for production operations

**合规判定**: ✅ PASS if monitoring solution detected, ❌ FAIL if none found

---

### **A14: Use Centralized Logging**
**目的**: Verifies log aggregation and centralized logging infrastructure

**检查内容**:
- Looks for Fluentd/Fluent Bit log collectors
- Detects Elasticsearch/OpenSearch backends
- Identifies CloudWatch Logs integration
- Checks for Loki logging stack

**重要性**:
- Centralizes logs from all cluster components
- Enables log analysis and troubleshooting
- Supports compliance and audit requirements
- Essential for distributed system debugging

**合规判定**: ✅ PASS if logging solution detected, ❌ FAIL if none found

---

### **C1: Monitor Control Plane Logs**
**目的**: Ensures EKS control plane logging is enabled for visibility and troubleshooting

**检查内容**:
- Uses AWS EKS API to check cluster logging configuration
- Verifies if 'api' log type is enabled in CloudWatch
- Checks control plane logging status

**重要性**:
- Provides visibility into cluster operations
- Essential for troubleshooting authentication and authorization issues
- Required for security auditing and compliance
- Helps identify performance bottlenecks

**合规判定**: ✅ PASS if control plane logging enabled, ❌ FAIL if disabled

---

### **C2: Cluster Authentication**
**目的**: Verifies proper authentication mechanisms are configured

**检查内容**:
- Checks for EKS Access Entries (modern API-based method)
- Falls back to aws-auth ConfigMap (traditional method)
- Verifies authentication configuration exists and is properly set up

**重要性**:
- Controls who can access the cluster
- EKS Access Entries provide better security than ConfigMap
- Essential for cluster security and access control
- Required for multi-user environments

**合规判定**: ✅ PASS if either authentication method properly configured, ❌ FAIL if neither found

---

### **C3: Running Large Clusters**
**目的**: Identifies large clusters (>1000 services) and checks for scale optimizations

**检查内容**:
- Counts total services in the cluster
- If >1000 services, checks for:
  - kube-proxy IPVS mode (better than iptables at scale)
  - AWS VPC CNI IP caching (WARM_IP_TARGET setting)

**重要性**:
- Large clusters face performance challenges with default settings
- iptables mode becomes inefficient with many services
- IP caching prevents EC2 API throttling
- Critical for maintaining performance at scale

**合规判定**:
- ✅ PASS if <1000 services (no optimization needed)
- ✅ PASS if >1000 services with proper optimizations
- ❌ FAIL if >1000 services without optimizations

---

### **C4: EKS Control Plane Endpoint Access Control**
**目的**: Ensures API server endpoint access is properly restricted

**检查内容**:
- Examines cluster endpoint access configuration
- Checks public/private access settings
- Verifies CIDR restrictions on public access
- Flags unrestricted public access (0.0.0.0/0)

**重要性**:
- Prevents unauthorized access to cluster API
- Reduces attack surface
- Best practice for production clusters
- Required for security compliance

**合规判定**: ✅ PASS if access properly restricted, ❌ FAIL if unrestricted public access

---

### **C5: Avoid Catch-All Admission Webhooks**
**目的**: Identifies overly broad admission webhooks that could impact performance

**检查内容**:
- Scans MutatingWebhookConfiguration and ValidatingWebhookConfiguration
- Identifies webhooks with:
  - Missing namespace/object selectors
  - Wildcard (*) in API groups, versions, or resources
  - Overly permissive scope settings

**重要性**:
- Catch-all webhooks intercept ALL matching requests
- Can cause significant performance degradation
- May lead to unexpected behavior
- Difficult to troubleshoot

**合规判定**: ✅ PASS if no catch-all webhooks found, ❌ FAIL if overly broad webhooks detected

---

### **D1: Use Kubernetes Cluster Autoscaler or Karpenter**
**目的**: Ensures automatic node scaling capability is available

**检查内容**:
- Looks for Cluster Autoscaler deployment
- Checks for Karpenter installation (namespace, deployments, CRDs)
- Verifies at least one node autoscaling solution exists

**重要性**:
- Automatically scales worker nodes based on demand
- Prevents resource shortages during traffic spikes
- Reduces costs by scaling down unused nodes
- Essential for dynamic workloads

**合规判定**: ✅ PASS if either solution found, ❌ FAIL if neither exists

---

### **D2: Worker Nodes Spread Across Multiple AZs**
**目的**: Ensures high availability through multi-AZ node distribution

**检查内容**:
- Examines all worker nodes
- Counts nodes per availability zone using node labels
- Checks for balanced distribution across AZs (within 20% variance)

**重要性**:
- Protects against AZ-level failures
- Improves application availability
- Better resource distribution
- Required for production workloads

**合规判定**:
- ✅ PASS if nodes spread across multiple AZs with balanced distribution
- ❌ FAIL if single AZ or uneven distribution

---

### **D3: Configure Resource Requests/Limits**
**目的**: Ensures all deployments have proper resource constraints

**检查内容**:
- Examines all Deployments in target namespaces
- Verifies each container has CPU and memory requests AND limits
- Identifies deployments without complete resource specifications

**重要性**:
- Prevents resource starvation and noisy neighbor problems
- Enables proper scheduling decisions
- Required for Quality of Service guarantees
- Essential for cluster stability

**合规判定**: ✅ PASS if all deployments have complete resource specs, ❌ FAIL if any are missing

---

### **D4: Namespace ResourceQuotas**
**目的**: Ensures resource governance is in place for namespaces

**检查内容**:
- Focuses on `default` namespace and user-created namespaces
- Ignores system namespaces (`kube-system`, `kube-public`, `kube-node-lease`)
- Verifies ResourceQuota objects exist for target namespaces

**重要性**:
- Prevents resource abuse and overconsumption
- Enables multi-tenancy
- Controls cluster resource allocation
- Required for production environments

**合规判定**: ✅ PASS if all target namespaces have ResourceQuotas, ❌ FAIL if any are missing

---

### **D5: Namespace LimitRanges**
**目的**: Ensures default resource limits are configured for namespaces

**检查内容**:
- Focuses on `default` namespace and user-created namespaces
- Ignores system namespaces (`kube-system`, `kube-public`, `kube-node-lease`)
- Verifies LimitRange objects exist for target namespaces

**重要性**:
- Provides default resource limits for containers
- Prevents containers from consuming unlimited resources
- Complements ResourceQuotas for complete resource governance
- Essential for cluster stability

**合规判定**: ✅ PASS if all target namespaces have LimitRanges, ❌ FAIL if any are missing

---

### **D6: Monitor CoreDNS Metrics**
**目的**: Ensures DNS service monitoring is configured

**检查内容**:
- Verifies CoreDNS deployment exists with metrics port (9153)
- Looks for ServiceMonitor (Prometheus Operator) or scrape configs
- Checks for DNS monitoring infrastructure

**重要性**:
- DNS is critical for cluster functionality
- Monitoring helps detect DNS performance issues
- Essential for troubleshooting connectivity problems
- Required for production operations

**合规判定**: ✅ PASS if CoreDNS metrics are monitored, ❌ FAIL if monitoring missing

---

### **D7: CoreDNS Configuration**
**目的**: Verifies CoreDNS is properly managed and configured

**检查内容**:
- For EKS auto mode clusters: Always passes (managed automatically)
- For regular clusters: Checks if CoreDNS is managed by EKS Managed Add-on
- Verifies CoreDNS deployment exists

**重要性**:
- EKS Managed Add-ons provide automatic updates and security patches
- Better integration with EKS platform
- Simplified management and maintenance
- Improved security posture

**合规判定**:
- ✅ PASS for auto mode clusters (always managed)
- ✅ PASS if CoreDNS is EKS managed add-on
- ❌ FAIL if CoreDNS is self-managed in regular clusters

---

## 总结

The EKS Resiliency Checker provides a comprehensive evaluation of cluster health across 26 critical areas. Each check includes:

- **Clear compliance status** (✅ PASS / ❌ FAIL)
- **Detailed findings** with specific resources identified
- **Actionable remediation guidance** with code examples
- **Contextual explanations** of why each check matters

The tool helps ensure EKS clusters follow best practices for:
- **High Availability**: Multi-replica deployments, anti-affinity, PDBs
- **Scalability**: Autoscaling, resource management, large cluster optimizations
- **Observability**: Monitoring, logging, metrics collection
- **Security**: Access control, endpoint restrictions, authentication
- **Operational Excellence**: Resource governance, graceful termination, service mesh

This comprehensive approach helps teams build and maintain resilient, production-ready EKS clusters.
