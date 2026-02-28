# External Secrets - Production-Ready Helm Chart

<p><img src="https://raw.githubusercontent.com/external-secrets/external-secrets/main/assets/eso-logo-large.png" width="100x" alt="external-secrets"></p>

![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![Version: 2.0.1](https://img.shields.io/badge/Version-2.0.1-informational?style=flat-square)

External secrets management for Kubernetes - configured with **production-ready defaults** for cloud-agnostic deployment.

## Overview

This chart deploys the [External Secrets Operator](https://external-secrets.io/) with hardened production defaults. It runs on **any Kubernetes cluster** (on-prem, AWS, Azure, GCP, or any other cloud) without requiring cloud-specific configurations.

### Components Deployed

| Component | Purpose | Default Replicas |
|-----------|---------|-----------------|
| **Operator (Controller)** | Watches ExternalSecret/SecretStore resources and syncs secrets from external providers | 3 |
| **Webhook** | Validates ExternalSecret and SecretStore manifests via admission control | 3 |
| **Cert Controller** | Manages TLS certificates for the webhook (when not using cert-manager) | 3 |

## TL;DR

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -f values.yaml
```

## Installing the Chart

```bash
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  -f values.yaml
```

## Production Changes Summary

Below is a detailed explanation of every change made to `values.yaml` and **why** each change matters for production.

---

### 1. Replica Count: 1 -> 3 (All Components)

**What changed:**
```yaml
# Main operator
replicaCount: 3

# Webhook
webhook:
  replicaCount: 3

# Cert Controller
certController:
  replicaCount: 3
```

**Why:** A single replica is a single point of failure. With 3 replicas per component:
- The **operator** uses leader election, so one replica is active and the others are hot standby. If the leader dies, another replica takes over within seconds.
- The **webhook** actively load-balances across all 3 replicas, so Kubernetes API admission requests are served even if one pod goes down.
- The **cert controller** ensures TLS certificate management continues uninterrupted during node failures or rolling updates.

---

### 2. Leader Election: false -> true

**What changed:**
```yaml
leaderElect: true
```

**Why:** With multiple operator replicas, leader election is **required** to prevent duplicate reconciliation. Without it, all 3 replicas would independently reconcile every ExternalSecret, causing unnecessary API calls to secret providers and potential rate limiting. Leader election ensures exactly one active controller at a time, with automatic failover.

---

### 3. Concurrent Reconciliations: 1 -> 5

**What changed:**
```yaml
concurrent: 5
```

**Why:** The default of 1 means ExternalSecrets are reconciled one at a time sequentially. In production clusters with hundreds or thousands of ExternalSecrets, this creates a bottleneck. Setting this to 5 allows the operator to process 5 ExternalSecrets simultaneously, significantly reducing sync latency. Adjust higher if you have thousands of secrets.

---

### 4. Resource Requests and Limits (All Components)

**What changed:**

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|------------|-----------|----------------|--------------|
| **Operator** | 100m | 500m | 256Mi | 512Mi |
| **Webhook** | 100m | 500m | 256Mi | 512Mi |
| **Cert Controller** | 50m | 200m | 128Mi | 256Mi |

```yaml
# Main operator
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Webhook
webhook:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

# Cert Controller
certController:
  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
```

**Why:**
- **Without requests**, the Kubernetes scheduler has no information about pod resource needs, leading to over-scheduling nodes and potential OOM kills or CPU starvation.
- **Without limits**, a runaway pod can consume all node resources, impacting other workloads.
- The cert controller gets lower allocations because it only manages TLS certificates and is less resource-intensive.
- These values are starting points - monitor actual usage with Prometheus and adjust accordingly.

---

### 5. Pod Disruption Budgets: Disabled -> Enabled (All Components)

**What changed:**
```yaml
# Main operator
podDisruptionBudget:
  enabled: true
  minAvailable: 1

# Webhook
webhook:
  podDisruptionBudget:
    enabled: true
    minAvailable: 1

# Cert Controller
certController:
  podDisruptionBudget:
    enabled: true
    minAvailable: 1
```

**Why:** PDBs prevent Kubernetes from voluntarily evicting all pods at once during operations like node drains, cluster upgrades, or autoscaler scale-downs. With `minAvailable: 1`, at least one pod of each component is always running, ensuring:
- Secrets continue to sync during cluster maintenance
- Webhook validation remains available (preventing API server errors)
- Certificate management is not interrupted

---

### 6. Topology Spread Constraints (All Components)

**What changed:**
```yaml
# Applied globally and per-component
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    matchLabelKeys:
      - pod-template-hash
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    matchLabelKeys:
      - pod-template-hash
```

**Why:** This is **cloud-agnostic** pod distribution:
- **Zone spread** (`topology.kubernetes.io/zone`): Distributes pods evenly across availability zones. Uses `ScheduleAnyway` so pods still schedule even if perfect balance isn't achievable (e.g., fewer zones than replicas).
- **Host spread** (`kubernetes.io/hostname`): Prevents multiple pods of the same component from landing on the same node. Uses `DoNotSchedule` because co-locating on the same node defeats the purpose of multiple replicas.
- These labels are standard Kubernetes labels that work on **any cluster** (AWS, Azure, GCP, on-prem) without modification.

---

### 7. Priority Class: "" -> system-cluster-critical (All Components)

**What changed:**
```yaml
# Main operator
priorityClassName: "system-cluster-critical"

# Webhook
webhook:
  priorityClassName: "system-cluster-critical"

# Cert Controller
certController:
  priorityClassName: "system-cluster-critical"
```

**Why:** External secrets management is critical infrastructure. Without a priority class, the operator pods could be preempted by less important workloads when the cluster is under resource pressure. `system-cluster-critical` is a built-in Kubernetes priority class that ensures these pods:
- Are among the last to be evicted under resource pressure
- Can preempt lower-priority pods if needed
- Are scheduled before regular workloads

---

### 8. Liveness Probe: Disabled -> Enabled (Operator)

**What changed:**
```yaml
livenessProbe:
  enabled: true
  spec:
    port: 8082
    timeoutSeconds: 5
    failureThreshold: 5
    periodSeconds: 10
    successThreshold: 1
    initialDelaySeconds: 10
    httpGet:
      port: live
      path: /healthz
```

**Why:** Without a liveness probe, Kubernetes cannot detect if the operator process is deadlocked or hung. The pod would remain "Running" but not actually processing secrets. With the probe enabled, Kubernetes automatically restarts unhealthy pods.

---

### 9. Startup Probe: Disabled -> Enabled (Cert Controller)

**What changed:**
```yaml
certController:
  startupProbe:
    enabled: true
    useReadinessProbePort: true
```

**Why:** The cert controller may take time to initialize on first startup (loading CRDs, generating initial certificates). A startup probe gives the container time to start before liveness checks kick in, preventing premature restarts.

---

### 10. Note on Persistent Storage (PVCs)

This chart deploys a **stateless Kubernetes operator**. All three components (operator, webhook, cert controller) are stateless - they read desired state from Kubernetes CRDs and fetch secrets from external providers at runtime. **No persistent storage is required.**

- Secret data is stored as native Kubernetes Secrets (managed by etcd)
- Configuration is stored as Kubernetes CRDs (ExternalSecret, SecretStore, etc.)
- TLS certificates are stored as Kubernetes Secrets

Therefore, no PVCs or storage classes have been added, as they would serve no purpose for this workload.

---

## Security Defaults (Unchanged - Already Production-Ready)

The chart already ships with strong security defaults that were preserved:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
  seccompProfile:
    type: RuntimeDefault
```

These ensure:
- Containers run as non-root (UID 1000)
- No privilege escalation is possible
- All Linux capabilities are dropped
- Root filesystem is read-only
- Seccomp profile restricts system calls

---

## Resource Tuning Guide

The default resource values are suitable for small-to-medium clusters. Adjust based on your workload:

| Cluster Size | ExternalSecrets Count | Recommended Operator CPU | Recommended Operator Memory | Concurrent |
|-------------|----------------------|--------------------------|----------------------------|------------|
| Small | < 100 | 100m / 500m | 256Mi / 512Mi | 5 |
| Medium | 100 - 500 | 200m / 1000m | 512Mi / 1Gi | 10 |
| Large | 500+ | 500m / 2000m | 1Gi / 2Gi | 20 |

## Configuration Reference

### Key Production Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `3` | Number of operator replicas |
| `leaderElect` | `true` | Enable leader election for HA |
| `concurrent` | `5` | Number of concurrent reconciliations |
| `priorityClassName` | `system-cluster-critical` | Pod priority class |
| `podDisruptionBudget.enabled` | `true` | Enable PDB |
| `podDisruptionBudget.minAvailable` | `1` | Minimum available pods during disruptions |
| `resources.requests.cpu` | `100m` | CPU request for operator |
| `resources.requests.memory` | `256Mi` | Memory request for operator |
| `resources.limits.cpu` | `500m` | CPU limit for operator |
| `resources.limits.memory` | `512Mi` | Memory limit for operator |
| `webhook.replicaCount` | `3` | Number of webhook replicas |
| `webhook.resources.requests.cpu` | `100m` | CPU request for webhook |
| `webhook.resources.requests.memory` | `256Mi` | Memory request for webhook |
| `certController.replicaCount` | `3` | Number of cert controller replicas |
| `certController.resources.requests.cpu` | `50m` | CPU request for cert controller |
| `certController.resources.requests.memory` | `128Mi` | Memory request for cert controller |

### Overriding Defaults

To override any production default, create a custom values file:

```yaml
# custom-values.yaml
replicaCount: 5
concurrent: 10
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

```bash
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  -f values.yaml \
  -f custom-values.yaml
```
