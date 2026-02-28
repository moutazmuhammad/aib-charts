# Harbor Helm Chart — Production-Ready Configuration

This is a production-hardened version of the [Harbor](https://goharbor.io) Helm chart (v1.18.2, app v2.14.2), configured for **cloud-agnostic** deployment on any Kubernetes cluster (on-prem, AWS, Azure, GCP, or any other cloud).

## Prerequisites

- Kubernetes cluster 1.20+
- Helm v3.2.0+
- A **default StorageClass** configured in the cluster (any provisioner works — cloud-managed, Longhorn, Rook-Ceph, NFS, etc.)

## What Was Changed and Why

### 1. Replica Counts — High Availability

All critical stateless components were scaled from **1 replica to 3 replicas** to ensure high availability and zero-downtime during rolling updates or node failures.

| Component    | Before | After | Why                                                              |
|-------------|--------|-------|------------------------------------------------------------------|
| **nginx**   | 1      | 3     | Reverse proxy — single point of entry; must survive node loss    |
| **portal**  | 1      | 3     | Web UI — user-facing; needs availability                         |
| **core**    | 1      | 3     | Central API server — most critical component in Harbor           |
| **jobservice** | 1   | 3     | Handles replication, GC, and scanning jobs; must stay available  |
| **registry** | 1     | 3     | Serves image pull/push; availability is essential for CI/CD      |
| **trivy**   | 1      | 3     | Vulnerability scanner — parallelizes scan load and stays resilient |
| **exporter** | 1     | 3     | Prometheus metrics — ensures monitoring continuity               |

> **Note:** Database (PostgreSQL) and Redis are StatefulSets with a single replica by design in this chart. For true HA on these, use external managed services (e.g., Amazon RDS, Azure Database for PostgreSQL, Redis Sentinel).

### 2. CPU and Memory Requests & Limits

Every component now has explicit resource **requests** (guaranteed resources) and **limits** (maximum allowed), preventing noisy-neighbor issues and enabling proper Kubernetes scheduling.

| Component              | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------------------|-------------|-----------|----------------|--------------|
| **nginx**             | 100m        | 500m      | 256Mi          | 512Mi        |
| **portal**            | 100m        | 500m      | 256Mi          | 512Mi        |
| **core**              | 200m        | 1         | 512Mi          | 1Gi          |
| **jobservice**        | 200m        | 1         | 512Mi          | 1Gi          |
| **registry**          | 200m        | 1         | 256Mi          | 1Gi          |
| **registryctl**       | 100m        | 500m      | 128Mi          | 256Mi        |
| **trivy**             | 200m        | 1         | 512Mi          | 1Gi          |
| **database** (PostgreSQL) | 200m   | 1         | 512Mi          | 2Gi          |
| **redis**             | 100m        | 500m      | 256Mi          | 512Mi        |
| **exporter**          | 100m        | 500m      | 128Mi          | 256Mi        |

**Why:** Without requests and limits, pods can be scheduled on overcommitted nodes and killed by the OOM killer under load. Setting these values ensures predictable performance and allows Kubernetes to make informed scheduling decisions.

### 3. Persistent Volume Claim (PVC) Sizes — Production Capacity

PVC sizes were increased from minimal defaults to production-appropriate values. The `storageClass` is left **empty** (`""`) so Kubernetes uses the cluster's **default StorageClass** — making this work on any cloud or on-prem environment without modification.

| PVC              | Before | After  | Why                                                                  |
|-----------------|--------|--------|----------------------------------------------------------------------|
| **registry**    | 5Gi    | 50Gi   | Stores container images — 5Gi fills up quickly in production         |
| **database**    | 1Gi    | 10Gi   | PostgreSQL metadata grows with projects, users, and audit logs       |
| **redis**       | 1Gi    | 5Gi    | Caching layer and job queues — needs headroom under load             |
| **jobservice**  | 1Gi    | 5Gi    | Job logs accumulate over time; prevents disk-full failures           |
| **trivy**       | 5Gi    | 10Gi   | Vulnerability DB cache grows with each update cycle                  |

**Cloud-Agnostic Design:** By not specifying a `storageClass`, the chart relies on the cluster's default. This means:
- On **AWS**: uses `gp3` (or whatever your default is)
- On **Azure**: uses `managed-premium` or `managed-csi`
- On **GCP**: uses `standard` or `premium-rwo`
- On **on-prem**: uses whatever default you configured (Longhorn, Rook-Ceph, NFS, etc.)

## What Was NOT Changed (and Why)

- **Exposure type** (`ingress`) — left as-is since Ingress is the standard cloud-agnostic way to expose services. Adjust `expose.ingress.hosts.core` and `externalURL` to match your domain.
- **TLS** (`certSource: auto`) — auto-generates self-signed certs by default. For production, you should switch to `certSource: secret` with cert-manager or your own certificates.
- **Admin password** (`Harbor12345`) — must be changed immediately after first login, or set via `existingSecretAdminPassword`.
- **Secret key** (`not-a-secure-key`) — must be replaced with a secure 16-character string for encryption at rest.
- **Internal TLS** (`enabled: false`) — can be enabled for inter-component encryption if required by your security policy.
- **Database and Redis types** (`internal`) — for true production HA, consider switching to external managed PostgreSQL and Redis services.
- **Update strategy** (`RollingUpdate`) — kept as-is, which is correct for production. Use `Recreate` only if your storage does not support `ReadWriteMany`.

## Quick Start

```bash
# Add the Harbor Helm repo (if not using local chart)
helm repo add harbor https://helm.goharbor.io
helm repo update

# Install with production values
helm install harbor . \
  --namespace harbor \
  --create-namespace \
  --set externalURL=https://harbor.yourdomain.com \
  --set expose.ingress.hosts.core=harbor.yourdomain.com \
  --set harborAdminPassword='YourSecurePassword' \
  --set secretKey='your-16-char-key'
```

## Post-Deployment Checklist

1. **Change the admin password** — default is `Harbor12345`
2. **Replace the secret key** — default is `not-a-secure-key`
3. **Configure TLS certificates** — switch from `auto` to cert-manager or manual certs
4. **Set up monitoring** — enable `metrics.enabled: true` and configure ServiceMonitor if using Prometheus
5. **Verify PVC provisioning** — run `kubectl get pvc -n harbor` to confirm all volumes are bound
6. **Consider external DB/Redis** — for true HA, use managed PostgreSQL and Redis services
