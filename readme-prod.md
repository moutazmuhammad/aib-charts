# Production Deployment Guide

Cloud-agnostic deployment of the full platform stack on any Kubernetes cluster
(on-prem, AWS, Azure, GCP, or any other provider).

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Cluster Requirements](#2-cluster-requirements)
3. [Namespace Setup](#3-namespace-setup)
4. [Secrets Preparation](#4-secrets-preparation)
5. [Deployment Order](#5-deployment-order)
6. [Step-by-Step Installation](#6-step-by-step-installation)
7. [Post-Deployment Verification](#7-post-deployment-verification)
8. [Configuration Reference](#8-configuration-reference)
9. [Day-2 Operations](#9-day-2-operations)

---

## 1. Prerequisites

| Tool      | Minimum Version | Purpose                        |
|-----------|-----------------|--------------------------------|
| kubectl   | 1.27+           | Cluster access                 |
| helm      | 3.12+           | Chart installation             |
| openssl   | any             | Generating secrets             |

Verify your tools:

```bash
kubectl version --client
helm version
```

---

## 2. Cluster Requirements

### Minimum Resources

| Component         | Nodes | CPU (total) | Memory (total) |
|-------------------|-------|-------------|----------------|
| Istio control plane | -   | 4 CPU       | 8 Gi           |
| Istio gateway     | -     | 2 CPU       | 2 Gi           |
| Kiali             | -     | 2 CPU       | 4 Gi           |
| Harbor            | -     | 8 CPU       | 16 Gi          |
| Nexus             | -     | 4 CPU       | 8 Gi           |
| Argo CD           | -     | 6 CPU       | 8 Gi           |
| External Secrets  | -     | 2 CPU       | 3 Gi           |
| OAuth2 Proxy      | -     | 2 CPU       | 2 Gi           |
| **Total**         | **3+**| **~30 CPU** | **~51 Gi**     |

A minimum of **3 worker nodes** is required for pod anti-affinity and topology
spread constraints to function correctly.

### Storage

All charts use `storageClass: ""` which resolves to the **cluster default
StorageClass**. Ensure a default StorageClass exists:

```bash
kubectl get storageclass
# Verify one has (default) next to it
```

If no default exists, create one or set it:

```bash
kubectl patch storageclass <your-sc> \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Ingress

The istio-ingress gateway uses `Service type: LoadBalancer`. On bare-metal or
on-prem clusters without a cloud load balancer, install MetalLB or similar
first:

```bash
# Example: MetalLB (adjust IP range to your network)
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

---

## 3. Namespace Setup

Create the required namespaces before installing any chart:

```bash
kubectl create namespace istio-system
kubectl create namespace kiali-operator
kubectl create namespace harbor
kubectl create namespace nexus
kubectl create namespace argocd
kubectl create namespace external-secrets
kubectl create namespace oauth2-proxy
```

---

## 4. Secrets Preparation

Several charts require secrets to be set **before deployment**. Never commit
real credentials to Git. Use one of:

- Kubernetes Secrets created manually (shown below)
- External Secrets Operator (after it is deployed)
- Sealed Secrets / SOPS

### 4.1 Harbor Secrets

The following values in `harbor/values.yaml` **must be changed** before deploy:

| Key                             | Default Placeholder        | Requirement                |
|---------------------------------|----------------------------|----------------------------|
| `harborAdminPassword`           | `CHANGE_ME_BEFORE_DEPLOY`  | Strong password            |
| `secretKey`                     | `CHANGE_ME_16CHARS`        | Exactly 16 characters      |
| `database.internal.password`    | `CHANGE_ME_BEFORE_DEPLOY`  | Strong password            |
| `registry.credentials.username` | `harbor_registry_user`     | Change for production      |
| `registry.credentials.password` | `harbor_registry_password` | Strong password            |

Generate secrets:

```bash
# Generate a 16-char secret key for Harbor
openssl rand -hex 8

# Generate strong passwords
openssl rand -base64 24
```

### 4.2 OAuth2 Proxy Secrets

The following values in `oauth2-proxy/values.yaml` **must be configured** with
your identity provider (Keycloak, Azure AD, Google, GitHub, etc.):

| Key               | Default Placeholder    | Requirement                         |
|-------------------|------------------------|-------------------------------------|
| `config.clientID`     | `XXXXXXX`          | OAuth2 Client ID from your IdP      |
| `config.clientSecret` | `XXXXXXXX`         | OAuth2 Client Secret from your IdP  |
| `config.cookieSecret` | `XXXXXXXXXXXXXXXX` | 32-byte base64 random string        |

Generate a cookie secret:

```bash
openssl rand -base64 32 | head -c 32 | base64
```

### 4.3 Harbor Ingress Hostname

Update the Harbor ingress hostname to match your domain:

| Key                           | Default                | Action              |
|-------------------------------|------------------------|---------------------|
| `expose.ingress.hosts.core`   | `core.harbor.domain`   | Set your FQDN      |
| `externalURL`                 | `https://core.harbor.domain` | Match the above |

### 4.4 Argo CD Domain

Update the Argo CD domain:

| Key             | Default                | Action              |
|-----------------|------------------------|---------------------|
| `global.domain` | `argocd.example.com`  | Set your FQDN      |

---

## 5. Deployment Order

Charts must be deployed in dependency order. The Istio service mesh must be
running before workloads that depend on sidecar injection.

```
Phase 1 - Service Mesh Foundation
  1. istio-base      (CRDs and cluster-wide resources)
  2. istiod          (control plane)
  3. istio-ingress   (gateway)
  4. kiali-operator  (mesh observability)

Phase 2 - Secrets Management
  5. external-secrets (operator for syncing secrets from vaults)

Phase 3 - Authentication
  6. oauth2-proxy    (SSO proxy)

Phase 4 - Platform Services
  7. argo-cd         (GitOps CD)
  8. harbor          (container registry)
  9. nexus           (artifact repository)
```

---

## 6. Step-by-Step Installation

### Phase 1: Service Mesh

#### 1. Istio Base (CRDs)

```bash
helm upgrade --install istio-base ./base \
  --namespace istio-system \
  --wait
```

#### 2. Istiod (Control Plane)

```bash
helm upgrade --install istiod ./istiod \
  --namespace istio-system \
  --wait --timeout 5m
```

Verify istiod is running with 2 replicas:

```bash
kubectl get pods -n istio-system -l app=istiod
```

#### 3. Istio Ingress Gateway

```bash
helm upgrade --install istio-ingress ./gateway \
  --namespace istio-system \
  --wait --timeout 5m
```

Verify the gateway has an external IP/hostname:

```bash
kubectl get svc -n istio-system istio-ingress
```

#### 4. Kiali Operator

```bash
helm upgrade --install kiali-operator ./kiali-operator \
  --namespace kiali-operator \
  --wait --timeout 5m
```

### Phase 2: Secrets Management

#### 5. External Secrets Operator

```bash
helm upgrade --install external-secrets ./external-secrets \
  --namespace external-secrets \
  --wait --timeout 5m
```

After deployment, create a `ClusterSecretStore` or `SecretStore` pointing to
your secrets backend (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault,
GCP Secret Manager, etc.):

```yaml
# Example: HashiCorp Vault ClusterSecretStore
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
```

### Phase 3: Authentication

#### 6. OAuth2 Proxy

Ensure you have set the OAuth2 secrets (section 4.2) first.

```bash
helm upgrade --install oauth2-proxy ./oauth2-proxy \
  --namespace oauth2-proxy \
  --wait --timeout 5m
```

### Phase 4: Platform Services

#### 7. Argo CD

```bash
helm upgrade --install argocd ./argo-cd \
  --namespace argocd \
  --wait --timeout 10m
```

Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

#### 8. Harbor

Ensure you have set Harbor secrets (section 4.1) and hostname (section 4.3).

```bash
helm upgrade --install harbor ./harbor \
  --namespace harbor \
  --wait --timeout 10m
```

#### 9. Nexus Repository Manager

```bash
helm upgrade --install nexus ./nexus-repository-manager \
  --namespace nexus \
  --wait --timeout 10m
```

Retrieve the initial admin password (if `NEXUS_SECURITY_RANDOMPASSWORD=true`):

```bash
kubectl exec -n nexus deploy/nexus-nexus-repository-manager -- \
  cat /nexus-data/admin.password 2>/dev/null
```

---

## 7. Post-Deployment Verification

### Health Checks

Run these commands to verify all components are healthy:

```bash
# Istio control plane
kubectl get pods -n istio-system
istioctl version  # if istioctl is installed

# Kiali
kubectl get pods -n kiali-operator
kubectl get kiali -A

# External Secrets
kubectl get pods -n external-secrets

# OAuth2 Proxy
kubectl get pods -n oauth2-proxy

# Argo CD
kubectl get pods -n argocd

# Harbor
kubectl get pods -n harbor

# Nexus
kubectl get pods -n nexus
```

### Verify HA Setup

Confirm pods are spread across nodes:

```bash
# Check pod distribution across nodes
kubectl get pods -A -o wide | grep -E "istio-system|argocd|harbor" | \
  awk '{print $1, $2, $8}' | sort
```

### Verify PodDisruptionBudgets

```bash
kubectl get pdb -A
```

All critical services should show a PDB.

### Verify Istio Mesh

```bash
# Check istiod health
kubectl get pods -n istio-system -l app=istiod

# Check mesh configuration
kubectl get configmap istio -n istio-system -o yaml | \
  grep -A5 "outboundTrafficPolicy"

# Verify access logging is enabled
kubectl get configmap istio -n istio-system -o yaml | \
  grep accessLogFile
```

---

## 8. Configuration Reference

### Chart Versions

| Chart                    | Chart Version | App Version |
|--------------------------|---------------|-------------|
| istio-base               | 1.29.0        | 1.29.0      |
| istiod                   | 1.29.0        | 1.29.0      |
| istio-ingress (gateway)  | 1.29.0        | 1.29.0      |
| kiali-operator           | 2.22.0        | v2.22.0     |
| harbor                   | 1.18.2        | 2.14.2      |
| nexus-repository-manager | 64.2.0        | 3.64.0      |
| argo-cd                  | 9.4.5         | v3.3.2      |
| external-secrets         | 2.0.1         | v2.0.1      |
| oauth2-proxy             | 10.1.4        | 7.14.2      |

### Production Defaults Per Chart

| Chart              | Replicas | PDB | Topology Spread | Anti-Affinity | Security Context | Priority Class         |
|--------------------|----------|-----|-----------------|---------------|------------------|------------------------|
| istiod             | 2 (HPA)  | Yes | Zone            | Hostname      | SeccompProfile   | -                      |
| istio-ingress      | 2 (HPA)  | Yes | Zone            | Hostname      | -                | -                      |
| kiali-operator     | 3        | Yes | Hostname        | Hostname      | Full hardened    | -                      |
| kiali-server (CR)  | 3        | -   | -               | Hostname      | -                | -                      |
| harbor (per svc)   | 3        | -   | Hostname        | -             | Full hardened    | -                      |
| nexus              | 1*       | Yes | Zone            | -             | UID/GID locked   | -                      |
| argo-cd (per svc)  | 3        | Yes | Zone (global)   | Hard (global) | Full hardened    | system-cluster-critical|
| external-secrets   | 3        | Yes | Zone + Hostname | -             | Full hardened    | system-cluster-critical|
| oauth2-proxy       | 3        | Yes | Zone            | Hostname      | Full hardened    | -                      |

*Nexus OSS uses an embedded DB and does not support multiple replicas.

### Storage Volumes

| Chart           | Volume                    | Default Size | Access Mode   |
|-----------------|---------------------------|--------------|---------------|
| Harbor          | registry                  | 50 Gi        | ReadWriteOnce |
| Harbor          | jobservice log            | 5 Gi         | ReadWriteOnce |
| Harbor          | database                  | 10 Gi        | ReadWriteOnce |
| Harbor          | redis                     | 5 Gi         | ReadWriteOnce |
| Harbor          | trivy                     | 10 Gi        | ReadWriteOnce |
| Nexus           | nexus-data                | 50 Gi        | ReadWriteOnce |
| Argo CD redis-ha| data                      | 10 Gi        | ReadWriteOnce |
| OAuth2 redis-ha | data                      | 10 Gi        | ReadWriteOnce |

All volumes use `storageClass: ""` (cluster default). No cloud-specific
storage classes are hardcoded.

---

## 9. Day-2 Operations

### Scaling

All HPA-enabled services (istiod, istio-ingress) auto-scale based on CPU.
For static-replica services, update the replica count in `values.yaml` and
run `helm upgrade`.

### Upgrading Charts

```bash
# Example: upgrading istiod
helm upgrade istiod ./istiod \
  --namespace istio-system \
  --wait --timeout 5m
```

Always upgrade Istio components in order: `base` -> `istiod` -> `gateway`.

### Backup

- **Harbor**: Back up the database PVC and registry PVC. Use Harbor's built-in
  replication to mirror images to another registry.
- **Nexus**: Back up the `nexus-data` PVC. Use Nexus blob store backup tasks.
- **Argo CD**: Configuration is in Git by design. Back up the `argocd` namespace
  secrets (repo credentials, cluster credentials).

### Monitoring

Istio, Harbor, and OAuth2 Proxy expose Prometheus metrics. To scrape them:

- Istio: metrics are auto-merged into application pods via
  `enablePrometheusMerge: true`
- Harbor: metrics enabled on port 8001 for core, registry, jobservice, exporter
- OAuth2 Proxy: metrics on port 44180
- Argo CD: enable `metrics.enabled: true` per component when a Prometheus
  Operator is available

### Disaster Recovery

1. Re-create namespaces (section 3)
2. Re-create secrets (section 4)
3. Deploy charts in order (section 6)
4. Restore PVC data from backups

Since Argo CD is deployed, after initial bootstrap you can manage all other
charts as Argo CD Applications for fully automated GitOps reconciliation.

---

## Quick Start (One-Liner per Phase)

```bash
# Phase 1 - Service Mesh
helm upgrade --install istio-base ./base -n istio-system --create-namespace --wait && \
helm upgrade --install istiod ./istiod -n istio-system --wait --timeout 5m && \
helm upgrade --install istio-ingress ./gateway -n istio-system --wait --timeout 5m && \
helm upgrade --install kiali-operator ./kiali-operator -n kiali-operator --create-namespace --wait --timeout 5m

# Phase 2 - Secrets Management
helm upgrade --install external-secrets ./external-secrets -n external-secrets --create-namespace --wait --timeout 5m

# Phase 3 - Authentication
helm upgrade --install oauth2-proxy ./oauth2-proxy -n oauth2-proxy --create-namespace --wait --timeout 5m

# Phase 4 - Platform Services
helm upgrade --install argocd ./argo-cd -n argocd --create-namespace --wait --timeout 10m && \
helm upgrade --install harbor ./harbor -n harbor --create-namespace --wait --timeout 10m && \
helm upgrade --install nexus ./nexus-repository-manager -n nexus --create-namespace --wait --timeout 10m
```
