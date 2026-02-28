# aib-charts

I have the following Helm charts downloaded locally: istio (istio-base, istiod, istio-ingress), kiali-operator, harbor, nexus, argo-cd, external-secrets, oauth2-proxy. 

I want you to prepare them for *production-ready deployment* in a *cloud-agnostic way* so that they can run on *any Kubernetes cluster* (on-prem, AWS, Azure, GCP, or any other cloud). 

Please generate:

1. *Helm charts adjustments*
   - Set production-ready defaults:
     - replicaCount: at least 3 per critical pod
     - CPU and memory requests and limits
     - Persistent storage with generic PVCs (no cloud-specific storage class)
   - Istio charts should have:
     - Correct namespace (e.g., service-mesh)
     - Gateways, VirtualServices, DestinationRules configured for production
     - TLS placeholders (cert-manager compatible)
   - Kiali operator configured with proper auth strategy
   - All charts cloud-agnostic (no cloud-specific annotations, no cloud-specific storage class or settings)
   - External-Secrets configured with generic backend placeholders

2. *Environment-specific values.yaml*
   - For dev, preprod, and prod
   - Configure replicas, resources, storage, ingress hosts
   - Secrets placeholders (OAuth2 Proxy, External-Secrets, Harbor, Nexus)
   - Ingress / Istio hostnames placeholders that can be replaced per environment

3. *ArgoCD Application manifests*
   - One manifest per Helm chart
   - Proper dependency order:
     - foundation (namespaces, RBAC)
     - Service Mesh (istio-base → istiod → istio-ingress → kiali-operator)
     - Secret Management (external-secrets)
     - Authentication (oauth2-proxy)
     - CI/CD (argo-cd → harbor → nexus)
   - Automated sync policy:
     - prune: true
     - selfHeal: true
   - Target namespace and cluster generic for any cloud

4. *Output*
   - Updated Helm chart directories ready to deploy
   - values.yaml per environment
   - ArgoCD manifests ready to apply
   - Notes on any manual steps required (like filling real secrets)

The goal is that after this preparation, I can push the repository to Git, and ArgoCD will automatically deploy everything to any Kubernetes cluster (on-prem or cloud) with minimal manual intervention.



--------------------------


● All 6 new directories/files created. Here's a summary of everything that was generated:                                                                                                                          
                                                                                                                                                                                                                   
  ---                                                                                                                                                                                                              
  Summary of Generated Files (62 new files)                                                                                                                                                                        
                                                                                                                                                                                                                   
  New Helm Charts (3)                                                                                                                                                                                              
                                                                                                                                                                                                                   
  ┌──────────────────────────┬──────────────────────────────────────────────────────────────────────┐                                                                                                              
  │          Chart           │                               Purpose                                │                                                                                                              
  ├──────────────────────────┼──────────────────────────────────────────────────────────────────────┤
  │ foundation/              │ Namespaces + RBAC for all platform components                        │
  ├──────────────────────────┼──────────────────────────────────────────────────────────────────────┤
  │ istio-config/            │ Istio Gateway, VirtualService, DestinationRule with cert-manager TLS │
  ├──────────────────────────┼──────────────────────────────────────────────────────────────────────┤
  │ external-secrets-config/ │ ClusterSecretStore + ExternalSecret resources (Vault/AWS/Azure/GCP)  │
  └──────────────────────────┴──────────────────────────────────────────────────────────────────────┘

  Environment Values (36 files - 12 per environment)

  Each environment (dev, preprod, prod) has values for all 12 components:

  ┌──────────────────┬──────────────┬─────────┬───────────────┬──────────────────────────────────────────────────────────────┐
  │    Component     │ Dev Replicas │ Preprod │     Prod      │                       Key Differences                        │
  ├──────────────────┼──────────────┼─────────┼───────────────┼──────────────────────────────────────────────────────────────┤
  │ Istiod           │ 1            │ 2       │ 3             │ Prod: zone-spread, 4Gi RAM, priority=system-cluster-critical │
  ├──────────────────┼──────────────┼─────────┼───────────────┼──────────────────────────────────────────────────────────────┤
  │ Istio Gateway    │ 1            │ 2       │ 3 (HPA to 10) │ Prod: PDB=2, zone-spread                                     │
  ├──────────────────┼──────────────┼─────────┼───────────────┼──────────────────────────────────────────────────────────────┤
  │ ArgoCD           │ 1            │ 2       │ 3             │ Prod: JSON logs, admin disabled, HA Redis                    │
  ├──────────────────┼──────────────┼─────────┼───────────────┼──────────────────────────────────────────────────────────────┤
  │ Harbor (core)    │ 1            │ 2       │ 3             │ Prod: 500Gi registry, zone-spread                            │
  ├──────────────────┼──────────────┼─────────┼───────────────┼──────────────────────────────────────────────────────────────┤
  │ OAuth2 Proxy     │ 1            │ 2       │ 3             │ Prod: Redis session store, zone-spread                       │
  ├──────────────────┼──────────────┼─────────┼───────────────┼──────────────────────────────────────────────────────────────┤
  │ External Secrets │ 1            │ 2       │ 3             │ Prod: leader election, 1Gi memory                            │
  ├──────────────────┼──────────────┼─────────┼───────────────┼──────────────────────────────────────────────────────────────┤
  │ Kiali            │ 1            │ 2       │ 3             │ Auth: anonymous(dev) → token(preprod) → openid(prod)         │
  ├──────────────────┼──────────────┼─────────┼───────────────┼──────────────────────────────────────────────────────────────┤
  │ Nexus            │ 1            │ 1       │ 1             │ Single-instance (stateful); prod: 4G JVM, 500Gi storage      │
  └──────────────────┴──────────────┴─────────┴───────────────┴──────────────────────────────────────────────────────────────┘

  ArgoCD Application Manifests (13 files)

  Sync wave ordering with prune: true and selfHeal: true:

  Wave 1:  Foundation (namespaces, RBAC)
  Wave 2:  Istio Base (CRDs)
  Wave 3:  Istiod (control plane)
  Wave 4:  Istio Ingress Gateway
  Wave 5:  Kiali Operator
  Wave 6:  External Secrets Operator
  Wave 7:  External Secrets Config (ClusterSecretStore + secrets)
  Wave 8:  OAuth2 Proxy
  Wave 9:  ArgoCD (self-managed)
  Wave 10: Harbor
  Wave 11: Nexus
  Wave 12: Istio Config (routing - deploys last)

  Plus app-of-apps.yaml as the bootstrap root application.

  Cloud-Agnostic Design

  - No cloud-specific StorageClass (uses cluster default)
  - No cloud-specific annotations
  - Generic LoadBalancer (works with MetalLB or cloud LBs)
  - Vault as default secret backend (switchable to AWS/Azure/GCP)
  - PVC-based filesystem storage (no cloud object store dependency)

  Manual Steps Required (see DEPLOYMENT-NOTES.md)

  1. Replace REPO_URL, TARGET_REVISION, ENVIRONMENT in ArgoCD app manifests
  2. Replace example.com hostnames with your actual domain
  3. Set up secret backend (Vault/AWS/Azure/GCP) and create auth secret
  4. Populate secrets in your backend (Harbor, OAuth2, ArgoCD, Nexus passwords)
  5. Configure your OIDC provider URLs in oauth2-proxy and kiali values
  6. Bootstrap: install ArgoCD manually, then apply app-of-apps.yaml

