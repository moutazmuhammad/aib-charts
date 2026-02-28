# Production-Ready Configuration (Cloud-Agnostic)

This chart has been configured with production-ready defaults designed to run on **any Kubernetes cluster** — on-prem, AWS, Azure, GCP, or any other cloud provider — without modification.

> **Requirement:** You need at least 3 worker nodes across multiple availability zones for optimal pod distribution.

## Changes Summary

### 1. Replica Counts (High Availability)

All critical components run multiple replicas to eliminate single points of failure.

| Component | Replicas | Notes |
|---|---|---|
| Application Controller | 3 | With dynamic cluster distribution for automatic shard rebalancing |
| API Server | 3 | Handles UI and API traffic |
| Repo Server | 3 | CPU-intensive manifest generation |
| ApplicationSet Controller | 3 | Generates Application resources |
| Dex (SSO) | 3 | Authentication server |
| Notifications | 2 | Uses leader election; 2 replicas for failover (extra replicas are idle standby) |

**Template changes**: Dex and Notifications deployment templates were modified to support configurable replicas via `dex.replicas` and `notifications.replicas` (defaults to 1 if not set for backward compatibility).

### 2. Dynamic Cluster Distribution (Controller)

`controller.dynamicClusterDistribution` is set to `true`. When running multiple controller replicas, clusters are sharded across them. Dynamic distribution automatically rebalances shards when replicas are added/removed or one becomes unhealthy. Without this, multi-replica controllers would not properly distribute work.

### 3. CPU and Memory Requests/Limits

All components have production resource requests and limits defined (previously all were `resources: {}`).

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| Application Controller | 500m | 1 core | 512Mi | 1Gi |
| API Server | 250m | 500m | 256Mi | 512Mi |
| Repo Server | 250m | 1 core | 256Mi | 1Gi |
| ApplicationSet Controller | 200m | 500m | 256Mi | 512Mi |
| Dex | 50m | 100m | 64Mi | 128Mi |
| Notifications | 100m | 200m | 128Mi | 256Mi |
| Redis | 200m | 500m | 256Mi | 512Mi |
| Redis Secret-Init Job | 100m | 200m | 64Mi | 128Mi |

**Why these values:**
- **Requests** ensure pods get guaranteed resources and proper scheduling across nodes.
- **Limits** prevent runaway pods from starving other workloads.
- Controller and Repo Server get the most resources because they are the most CPU/memory intensive (reconciliation loops and manifest generation).
- Dex gets the least because it handles only authentication traffic.

### 4. PodDisruptionBudgets (PDBs)

All components have PDBs enabled with `maxUnavailable: 1`.

| Component | PDB Setting |
|---|---|
| Application Controller | maxUnavailable: 1 |
| API Server | maxUnavailable: 1 |
| Repo Server | maxUnavailable: 1 |
| ApplicationSet Controller | maxUnavailable: 1 |
| Dex | maxUnavailable: 1 |
| Redis | maxUnavailable: 1 |
| Notifications | maxUnavailable: 1 |

PDBs prevent Kubernetes from evicting too many pods simultaneously during node drains, cluster upgrades, or maintenance. `maxUnavailable: 1` ensures at least N-1 pods remain running at all times.

### 5. Redis: Single Node to HA with Persistent Storage

| Setting | Before | After |
|---|---|---|
| `redis.enabled` | `true` | `false` |
| `redis-ha.enabled` | `false` | `true` |
| `redis-ha.persistentVolume.enabled` | `false` | `true` |
| `redis-ha.persistentVolume.storageClass` | (not set) | `""` (empty = cluster default) |
| `redis-ha.persistentVolume.size` | (not set) | `10Gi` |

**Why:**
- Single-node Redis is a single point of failure. Redis HA deploys 3 Redis nodes with Sentinel for automatic failover.
- Persistence ensures Redis data survives pod restarts (previously data was lost on every restart).
- `storageClass: ""` uses the cluster's **default StorageClass**, making it cloud-agnostic — works on AWS EBS, Azure Disk, GCP PD, on-prem NFS/Ceph, etc. without modification.

### 6. Global Production Defaults

**Pod Anti-Affinity:** `soft` → `hard`

Hard anti-affinity ensures replicas of the same component are **never** scheduled on the same node. This prevents a single node failure from taking down multiple replicas of the same service.

**Topology Spread Constraints:** Zone-aware pod distribution

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
```

Distributes pods across availability zones for zone failure resilience. `ScheduleAnyway` (soft) is used instead of `DoNotSchedule` so that clusters with fewer zones or single-zone clusters still work without scheduling failures. The `topology.kubernetes.io/zone` label is a standard Kubernetes label set by all major cloud providers and can be manually added to on-prem nodes.

**Deployment Strategy:** Zero-downtime rolling updates

```yaml
deploymentStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 0
```

`maxUnavailable: 0` ensures no pods are terminated before new ones are ready, guaranteeing zero-downtime deployments. `maxSurge: 25%` allows extra pods to be created during rollout for a smooth transition.

## Files Modified

| File | Change |
|---|---|
| `values.yaml` | All configuration changes (replicas, resources, PDBs, Redis HA, global defaults) |
| `templates/dex/deployment.yaml` | Made replicas configurable via `dex.replicas` |
| `templates/argocd-notifications/deployment.yaml` | Made replicas configurable via `notifications.replicas` |
