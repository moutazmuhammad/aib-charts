# Nexus Repository Manager - Production-Ready Helm Chart

Cloud-agnostic Helm chart for deploying Sonatype Nexus Repository Manager on any Kubernetes cluster (on-prem, AWS, Azure, GCP, or any other platform).

> **Note:** This chart is based on the original Sonatype Nexus Repository Manager Helm chart (v64.2.0, appVersion 3.64.0), which was deprecated by Sonatype in October 2023. It has been hardened with production-ready defaults for cloud-agnostic deployments.

## Production-Ready Changes

### 1. Replica Count

**What changed:** Made the replica count configurable via `replicaCount` (previously hardcoded to `1` in the Deployment template).

**Why it stays at 1:** Nexus Repository Manager OSS uses an embedded OrientDB database and file-based blob storage. It does **not** support running multiple replicas -- doing so would cause data corruption. If you need high availability, use Sonatype's official [HA/Resiliency Helm Chart](https://github.com/sonatype/nxrm3-ha-repository/tree/main/nxrm-ha) which requires a PostgreSQL database.

**What we did instead:** Added a `PodDisruptionBudget` with `minAvailable: 1` to protect the single replica during voluntary disruptions (node drains, cluster upgrades).

### 2. CPU and Memory Requests and Limits

**What changed:**
| Setting | Before (Default) | After (Production) |
|---------|-------------------|---------------------|
| CPU request | Not set | 4 cores |
| CPU limit | Not set | 4 cores |
| Memory request | Not set | 8Gi |
| Memory limit | Not set | 8Gi |
| JVM Heap (-Xms/-Xmx) | 2703M | 4G |
| MaxDirectMemorySize | 2703M | 4G |

**Why:** Without resource requests, the Kubernetes scheduler cannot make informed placement decisions, and the pod may be evicted under memory pressure. Without limits, a runaway process can starve other workloads. The values follow [Sonatype's system requirements](https://help.sonatype.com/repomanager3/product-information/system-requirements) for production instances. JVM heap was increased to match the higher memory allocation.

### 3. Persistent Storage (Cloud-Agnostic PVCs)

**What changed:**

| Setting | Before | After |
|---------|--------|-------|
| Storage size | 8Gi | 50Gi |
| StorageClass | Not set (cluster default) | Not set (cluster default) |
| GCP PersistentVolume template | Present (`gcePersistentDisk`) | Removed |

**Why:**
- **8Gi is too small** for production artifact storage. 50Gi provides a reasonable starting point; adjust based on your repository size.
- **Removed the GCP-specific `pv.yaml`** template that created a `PersistentVolume` with `gcePersistentDisk`. This made the chart tied to Google Cloud. By relying solely on PVCs with the cluster's default StorageClass, the chart now works on any platform:
  - **AWS**: Uses `gp2`/`gp3` via the EBS CSI driver
  - **GCP**: Uses `pd-standard`/`pd-ssd` via the GCE PD CSI driver
  - **Azure**: Uses `managed-premium` via the Azure Disk CSI driver
  - **On-prem**: Uses whatever StorageClass the cluster administrator has configured (e.g., Ceph, NFS, Longhorn)
- If you need a specific StorageClass, set `persistence.storageClass` in your values override.

### 4. PodDisruptionBudget (PDB)

**What changed:** Added a new `templates/pdb.yaml` template, enabled by default with `minAvailable: 1`.

**Why:** During voluntary disruptions (node drains, rolling cluster upgrades), Kubernetes will evict pods. Without a PDB, the Nexus pod can be evicted with no safeguards, causing downtime. With `minAvailable: 1`, Kubernetes will block a drain if it would leave zero Nexus pods running, giving the pod time to be rescheduled first.

### 5. Affinity and Topology Spread Constraints

**What changed:** Added support for `affinity` and `topologySpreadConstraints` in the Deployment template (both configurable via values).

**Why:** These allow operators to:
- Use `affinity` to control which nodes the Nexus pod is scheduled on (e.g., prefer nodes with SSD storage or specific hardware)
- Use `topologySpreadConstraints` to spread pods across failure domains (zones) -- useful if you ever move to the HA chart

### 6. Priority Class Support

**What changed:** Added `priorityClassName` support in the Deployment template.

**Why:** In resource-constrained clusters, a priority class ensures Nexus is not preempted by lower-priority workloads.

## Installation

```bash
helm install nexus . -n nexus --create-namespace
```

### Override values for your environment

```bash
helm install nexus . -n nexus --create-namespace \
  --set persistence.storageSize=100Gi \
  --set persistence.storageClass=my-storage-class
```

### Using a custom values file

```bash
helm install nexus . -n nexus --create-namespace -f my-values.yaml
```

## Key Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `1` | Number of replicas (must be 1 for OSS) |
| `nexus.resources.requests.cpu` | `4` | CPU request |
| `nexus.resources.requests.memory` | `8Gi` | Memory request |
| `nexus.resources.limits.cpu` | `4` | CPU limit |
| `nexus.resources.limits.memory` | `8Gi` | Memory limit |
| `persistence.enabled` | `true` | Enable persistent storage |
| `persistence.storageSize` | `50Gi` | PVC storage size |
| `persistence.storageClass` | _(unset, uses cluster default)_ | StorageClass name |
| `persistence.existingClaim` | _(unset)_ | Use an existing PVC |
| `podDisruptionBudget.enabled` | `true` | Enable PDB |
| `podDisruptionBudget.minAvailable` | `1` | Minimum available pods |
| `affinity` | `{}` | Pod affinity rules |
| `topologySpreadConstraints` | `[]` | Topology spread constraints |
| `tolerations` | `[]` | Pod tolerations |
| `ingress.enabled` | `false` | Enable Ingress |
| `service.type` | `ClusterIP` | Service type |

## Upgrading

```bash
helm upgrade nexus . -n nexus
```

## Uninstalling

```bash
helm uninstall nexus -n nexus
```

> **Warning:** By default, the PVC will remain after uninstall to prevent data loss. Delete it manually if you want to remove all data:
> ```bash
> kubectl delete pvc -n nexus -l app.kubernetes.io/name=nexus-repository-manager
> ```
