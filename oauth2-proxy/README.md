# oauth2-proxy - Production-Ready Helm Chart

[oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) is a reverse proxy and static file server that provides authentication using Providers (Google, GitHub, and others) to validate accounts by email, domain, or group.

This chart has been configured with **production-ready defaults** for cloud-agnostic deployment on any Kubernetes cluster (on-prem, AWS, Azure, GCP, or any other cloud).

## TL;DR

```console
helm install my-release ./oauth2-proxy
```

## Production-Ready Changes

The following changes were made to the default `values.yaml` to prepare this chart for production deployment.

### 1. High Availability - Replica Count

| Setting | Before | After |
|---------|--------|-------|
| `replicaCount` | `1` | `3` |

**Why:** A single replica is a single point of failure. Running 3 replicas ensures the authentication proxy remains available during node failures, rolling updates, and maintenance windows.

### 2. Resource Requests and Limits

**oauth2-proxy container:**

| Setting | Before | After |
|---------|--------|-------|
| `resources.requests.cpu` | _(unset)_ | `100m` |
| `resources.requests.memory` | _(unset)_ | `128Mi` |
| `resources.limits.cpu` | _(unset)_ | `200m` |
| `resources.limits.memory` | _(unset)_ | `256Mi` |

**waitForRedis init container:**

| Setting | Before | After |
|---------|--------|-------|
| `initContainers.waitForRedis.resources.requests.cpu` | _(unset)_ | `25m` |
| `initContainers.waitForRedis.resources.requests.memory` | _(unset)_ | `32Mi` |
| `initContainers.waitForRedis.resources.limits.cpu` | _(unset)_ | `50m` |
| `initContainers.waitForRedis.resources.limits.memory` | _(unset)_ | `64Mi` |

**Redis HA subchart resources:**

| Component | CPU Request/Limit | Memory Request/Limit |
|-----------|-------------------|----------------------|
| Redis | `100m` / `200m` | `128Mi` / `256Mi` |
| HAProxy | `100m` / `200m` | `128Mi` / `256Mi` |
| Sentinel | `50m` / `100m` | `64Mi` / `128Mi` |

**Why:** Without resource requests, the Kubernetes scheduler cannot make intelligent placement decisions. Without limits, a single pod can consume all node resources and cause cascading failures. These values are sized for a typical oauth2-proxy workload and should be adjusted based on observed usage.

### 3. Persistent Storage with Generic PVCs

| Setting | Before | After |
|---------|--------|-------|
| `redis-ha.enabled` | `false` | `true` |
| `redis-ha.replicas` | _(unset)_ | `3` |
| `redis-ha.persistentVolume.enabled` | _(unset)_ | `true` |
| `redis-ha.persistentVolume.size` | _(unset)_ | `10Gi` |
| `redis-ha.persistentVolume.storageClass` | _(unset)_ | `""` (cluster default) |
| `sessionStorage.type` | `cookie` | `redis` |

**Why:** Redis-backed session storage is more robust than cookie-based storage for production. It allows sessions to survive pod restarts and supports larger session data. The `storageClass` is set to `""` (empty string) which tells Kubernetes to use the cluster's default StorageClass - this is **cloud-agnostic** and works on any cluster (AWS EBS, Azure Disk, GCP PD, Ceph, NFS, local-path, etc.) without requiring any cloud-specific configuration.

### 4. Rolling Update Strategy

| Setting | Before | After |
|---------|--------|-------|
| `strategy.type` | _(unset)_ | `RollingUpdate` |
| `strategy.rollingUpdate.maxSurge` | _(unset)_ | `1` |
| `strategy.rollingUpdate.maxUnavailable` | _(unset)_ | `0` |

**Why:** Setting `maxUnavailable: 0` ensures zero-downtime deployments. Kubernetes will always bring up a new pod before terminating an old one during updates.

### 5. Pod Disruption Budget

| Setting | Before | After |
|---------|--------|-------|
| `podDisruptionBudget.maxUnavailable` | `null` | `1` |
| `podDisruptionBudget.minAvailable` | `1` | `null` |

**Why:** With 3 replicas, using `maxUnavailable: 1` allows cluster maintenance (node drains, upgrades) while guaranteeing at least 2 pods remain available at all times.

### 6. Pod Anti-Affinity

| Setting | Before | After |
|---------|--------|-------|
| `affinity` | _(unset)_ | Pod anti-affinity on `kubernetes.io/hostname` |

**Why:** Soft (preferred) pod anti-affinity spreads oauth2-proxy replicas across different nodes. This prevents all replicas from being lost if a single node goes down. Using `preferredDuringSchedulingIgnoredDuringExecution` ensures pods can still be scheduled on the same node if no other nodes are available.

### 7. Topology Spread Constraints

| Setting | Before | After |
|---------|--------|-------|
| `topologySpreadConstraints` | _(unset)_ | Spread across `topology.kubernetes.io/zone` |

**Why:** Distributes pods across availability zones for resilience against zone-level failures. Uses `whenUnsatisfiable: ScheduleAnyway` so it works gracefully on single-zone clusters too.

### 8. Pod Security Context

| Setting | Before | After |
|---------|--------|-------|
| `podSecurityContext` | `{}` | `runAsNonRoot`, `runAsUser/Group: 2000`, `fsGroup: 2000`, `seccompProfile: RuntimeDefault` |

**Why:** Pod-level security context complements the existing container-level security context. Setting `fsGroup` ensures any persistent volume mounts have the correct group ownership. The `seccompProfile: RuntimeDefault` restricts system calls at the pod level.

### 9. Graceful Shutdown

| Setting | Before | After |
|---------|--------|-------|
| `terminationGracePeriodSeconds` | _(unset, default 30)_ | `65` |
| `lifecycle.preStop` | _(unset)_ | `sleep 10` |

**Why:** The `preStop` hook gives the Kubernetes endpoints controller time to remove the pod from Service endpoints before it starts shutting down. This prevents traffic from being routed to a pod that is terminating. The `terminationGracePeriodSeconds` of 65 allows enough time for the 10-second sleep plus the application's graceful shutdown.

### 10. Redis HA Configuration

| Setting | Before | After |
|---------|--------|-------|
| `redis-ha.haproxy.enabled` | _(unset)_ | `true` |
| `redis-ha.haproxy.replicas` | _(unset)_ | `3` |
| `redis-ha.redis.config.min-replicas-to-write` | _(unset)_ | `1` |

**Why:** HAProxy provides a single stable endpoint for Redis access and handles master failover transparently. Setting `min-replicas-to-write: 1` ensures data is replicated to at least one replica before confirming writes, preventing data loss during failover.

## Cloud-Agnostic Design

This chart avoids any cloud-provider-specific configuration:

- **No cloud-specific StorageClass**: Uses `""` (empty string) to leverage the cluster's default StorageClass, which is provisioned by whichever CSI driver or storage provider is installed
- **No cloud-specific annotations**: No AWS ALB, GCP NEG, or Azure-specific annotations are set
- **No cloud-specific node selectors**: No cloud provider labels are used for scheduling
- **Standard Kubernetes APIs only**: Uses only stable, widely-supported Kubernetes APIs (apps/v1, policy/v1)
- **Generic topology keys**: Uses `kubernetes.io/hostname` and `topology.kubernetes.io/zone` which are standard across all Kubernetes distributions

## Pre-Deployment Checklist

Before deploying to production, ensure you:

1. **Set OAuth credentials**: Replace the placeholder values in `config.clientID`, `config.clientSecret`, and `config.cookieSecret` (or use `config.existingSecret` to reference a pre-created Secret)
2. **Configure your provider**: Update `config.configFile` with your OAuth provider settings
3. **Set a Redis password**: Configure `redis-ha.redisPassword` and ensure `sessionStorage.redis.password` matches
4. **Enable Ingress or Gateway API**: Configure `ingress` or `gatewayApi` based on your cluster's ingress controller
5. **Review resource limits**: Adjust CPU/memory based on your expected traffic load
6. **Enable monitoring**: Set `metrics.serviceMonitor.enabled: true` if you use Prometheus Operator

## Configuration

See `values.yaml` for the full list of configurable parameters.

## Uninstalling

```console
helm uninstall my-release
```

> **Note:** Persistent Volume Claims created by Redis HA are not automatically deleted. Clean them up manually if needed:
> ```console
> kubectl delete pvc -l app=redis-ha
> ```
