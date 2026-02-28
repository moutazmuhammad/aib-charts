# Kiali Operator Helm Chart - Production-Ready

Kiali is an open source project for service mesh observability, providing a management console for Istio service meshes. This chart deploys the Kiali Operator, which manages the lifecycle of Kiali Server instances via Custom Resources.

- **Chart Version**: 2.22.0
- **App Version**: v2.22.0

## Production-Readiness Changes

This chart has been hardened for production deployment across any Kubernetes cluster (on-prem, AWS, Azure, GCP, or any other cloud). Below is a summary of every change made and the reasoning behind it.

---

### 1. Replica Count Increased to 3

| Setting | Before | After |
|---------|--------|-------|
| `replicaCount` | `1` | `3` |
| `cr.spec.deployment.replicas` | `1` (default) | `3` |

**Why**: A single replica is a single point of failure. With 3 replicas, the operator remains available during node failures, rolling updates, and maintenance windows. The operator already supports leader election (`--leader-election-id`), so multiple replicas work safely - one active leader handles reconciliation while standbys are ready for immediate failover. The Kiali Server CR defaults were also updated to 3 replicas for the same reason.

---

### 2. CPU and Memory Requests and Limits

| Setting | Before | After |
|---------|--------|-------|
| Operator CPU request | `10m` | `50m` |
| Operator memory request | `64Mi` | `128Mi` |
| Operator CPU limit | *not set* | `500m` |
| Operator memory limit | *not set* | `512Mi` |
| Kiali Server CPU request (CR) | `10m` (default) | `50m` |
| Kiali Server memory request (CR) | `64Mi` (default) | `128Mi` |
| Kiali Server CPU limit (CR) | *not set* | `500m` |
| Kiali Server memory limit (CR) | *not set* | `1Gi` |

**Why**:
- **Requests** were increased from the bare minimum to realistic production values. The original 10m CPU / 64Mi memory was appropriate for development but insufficient under production reconciliation loads. The new values ensure the scheduler places pods on nodes with adequate capacity.
- **Limits** were added to prevent runaway resource consumption. Without limits, a memory leak or CPU spike in the operator could starve other workloads on the node. The limits are set at 10x the requests for CPU and 4x for memory, providing headroom for burst activity during reconciliation loops while protecting cluster stability.
- The Kiali Server CR defaults were also updated with production-grade resource definitions.

---

### 3. Persistent Storage with Generic PVCs

```yaml
persistence:
  enabled: false        # Set to true to enable
  storageClass: ""      # Empty = use cluster default (cloud-agnostic)
  accessMode: ReadWriteOnce
  size: 1Gi
```

**Why**: A PVC template has been added for optional persistent storage, mounted at `/data`. This can be used for caching operator data (ansible artifacts, reconciliation state) that should survive pod restarts.

**Cloud-agnostic design**:
- `storageClass: ""` (empty) means the cluster's **default StorageClass** is used automatically. This works on any Kubernetes distribution - AWS EBS, Azure Disk, GCP PD, Longhorn, Rook-Ceph, local-path, or any other provisioner.
- No cloud-specific annotations or provisioner references are hardcoded.
- Set `storageClass: "-"` to explicitly disable dynamic provisioning (for pre-provisioned PVs).
- Set `storageClass: "my-class"` to use a specific StorageClass by name.

**Note**: Persistence is **disabled by default** because the Kiali Operator is fundamentally stateless - it reads Kiali CRs and reconciles cluster state. The `/tmp` emptyDir volume remains for runtime scratch space. Enable persistence only if you need data to survive pod restarts for your specific use case.

---

### 4. PodDisruptionBudget (New)

```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

**Why**: A PodDisruptionBudget ensures that at least 1 operator pod remains running during voluntary disruptions (node drains, cluster upgrades, scaling events). Without a PDB, `kubectl drain` or a cluster autoscaler could evict all operator pods simultaneously, causing a service outage. With `replicaCount: 3` and `minAvailable: 1`, up to 2 pods can be disrupted at once while maintaining operator availability.

A new template file `templates/pdb.yaml` was created for this resource.

---

### 5. Topology Spread Constraints (New)

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
```

**Why**: Running all 3 operator replicas on the same node defeats the purpose of high availability. Topology spread constraints distribute pods evenly across nodes. If a node goes down, only one replica is lost.

- `topologyKey: kubernetes.io/hostname` spreads across nodes (works on every Kubernetes cluster).
- `whenUnsatisfiable: ScheduleAnyway` is a soft constraint - pods will still schedule if perfect distribution isn't possible (e.g., a 2-node cluster). This avoids unschedulable pods.
- The selector labels are automatically populated from the chart's standard selector labels in the deployment template.

---

### 6. Image Pull Policy Changed

| Setting | Before | After |
|---------|--------|-------|
| `image.pullPolicy` | `Always` | `IfNotPresent` |

**Why**: `Always` forces a registry pull on every pod start, adding latency and creating a hard dependency on registry availability. In production, `IfNotPresent` uses the cached image if it already exists on the node, which is faster, more reliable, and reduces egress costs. The image tag is version-pinned (`v2.22.0`), so there is no risk of running a stale image.

---

### 7. Debug Logging Disabled

| Setting | Before | After |
|---------|--------|-------|
| `debug.enabled` | `true` | `false` |

**Why**: Full ansible debug logs after every reconciliation loop generate excessive log volume in production. This increases storage costs, makes log analysis harder, and can impact performance. Debug logging should be enabled on-demand for troubleshooting, not left on permanently. Verbosity level `1` is retained for basic operational logging.

---

### 8. Security Defaults (Unchanged - Already Production-Grade)

The following security settings were already properly configured and remain unchanged:

- `runAsNonRoot: true` - Container cannot run as root
- `allowPrivilegeEscalation: false` - No privilege escalation
- `readOnlyRootFilesystem: true` - Filesystem is read-only
- `capabilities.drop: [ALL]` - All Linux capabilities dropped
- Health probes (readiness, liveness, startup) - Already configured
- Prometheus metrics scraping - Already enabled

---

## Installation

```bash
helm install kiali-operator ./kiali-operator \
  --namespace kiali-operator \
  --create-namespace
```

### With Kiali Server CR

```bash
helm install kiali-operator ./kiali-operator \
  --namespace kiali-operator \
  --create-namespace \
  --set cr.create=true
```

### With Persistent Storage Enabled

```bash
helm install kiali-operator ./kiali-operator \
  --namespace kiali-operator \
  --create-namespace \
  --set persistence.enabled=true
```

### With a Specific StorageClass

```bash
helm install kiali-operator ./kiali-operator \
  --namespace kiali-operator \
  --create-namespace \
  --set persistence.enabled=true \
  --set persistence.storageClass=my-storage-class
```

---

## Key Configuration Reference

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `3` | Number of operator pod replicas |
| `resources.requests.cpu` | `50m` | CPU request for operator pod |
| `resources.requests.memory` | `128Mi` | Memory request for operator pod |
| `resources.limits.cpu` | `500m` | CPU limit for operator pod |
| `resources.limits.memory` | `512Mi` | Memory limit for operator pod |
| `persistence.enabled` | `false` | Enable persistent storage |
| `persistence.storageClass` | `""` | StorageClass name (empty = cluster default) |
| `persistence.accessMode` | `ReadWriteOnce` | PVC access mode |
| `persistence.size` | `1Gi` | PVC storage size |
| `podDisruptionBudget.enabled` | `true` | Enable PDB |
| `podDisruptionBudget.minAvailable` | `1` | Minimum available pods during disruption |
| `topologySpreadConstraints` | spread across nodes | Pod distribution constraints |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `debug.enabled` | `false` | Enable full ansible debug logs |
| `cr.spec.deployment.replicas` | `3` | Kiali Server replica count |
| `cr.spec.deployment.resources` | see values.yaml | Kiali Server resource requests/limits |

---

## New Template Files

| File | Purpose |
|------|---------|
| `templates/pdb.yaml` | PodDisruptionBudget for high availability |
| `templates/pvc.yaml` | PersistentVolumeClaim for optional persistent storage |
