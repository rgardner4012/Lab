# Architectural Decissions

## 001 - Flux over argo

### Context

I want a gitOps engine to reconcile as much of the configuration as possible. Currently the 2 front runners are Flux and Argo CD.

### Decision

Use Flux

### Impacts

- Lower resource utilization.
- Flux natively supports SOPS integration vs requiring plugins or wrappers.
- For our single cluster the benefits of Argo's RBAC and multi-cluster UI provide no benefit.
- PR's will serve as the review mechanism.

## 002 - Traefik over ingress-nginx

### Context

We will need an ingress controller

### Decision

Use Traefik

### Impacts

- Ingress-nginx is being retired in the next 2 months. Even though we could still use it and benefit from it's extensive example-set it's not worth moving forward with it.
- Traefik uses the new gateway api, and is a leading implementation of it.
- Default for rke2, reducing friction.
- Native letsEncrypt.
- Dynamic configuration.
- Built in dashboard.
- Steeper learning curve.

## 003 - Secrets Strategy - SOPS & OpenBAO

### Context

We need to manage secrets for at least 2 platforms: K8s and Docker compose stacks. I want it secure and recoverable.

### Decision

OpenBao + External Secrets Operator for K8s secrets, SOPS for Docker secrets

### Impacts

- One Age key for both SOPS and Flux bootstrap.
- ESO Syncs k8s secrets dynamically. Also caches secrets allowing existing workloads to run if OpenBao is temporarily down.
- OpenBao demostrates production grade secrets management with a real secrets platform, complete with auditing and auth using service acct tokens.
- SOPS benefits: K8s might be more likely to be reconfigured, I'd like the nas docker services to not be reliant on any of the k8s services.
- SOPS, simple to use, single key, and git native, enabling rollbacks and versioning.

## 004 Longhorn and NFS for k8s storage options

### Context

I want ReadWriteMany functionality. With the Ugreen nas there was limited CSI capability. In general the k8s services don't require much storage.

### Decision

Longhorn enables local disk speed while allowing for ReadWriteMany functionality. NFS can cause issues, but gives me greater access to space, so it's a backup option if necessary.

### Impacts

- About 180Gb usable with 3 replicas in longhorn, maybe 3-400 if we drop to 2 replicas.
- Media workoads (*arr/plex) stay on Docker/NAS with local storage.
- Longhorn runs a storage engine per node, consuming more RAM/CPU.
- NFS may not ever be required.

## 005 - Media on Docker

### Context

The Media stack (Plex, Radarr, Sonarr, Prowlar, Configarr) requires significant amounts of storage, and was originally setup as docker compose stacks in portainer on the NAS. I need to decide if I want to migrate it to k8s.

### Decision

Keep it as Docker compose on the NAS. HA features of K8s, don't warrant lost functionality by moving it off the NAS.

### Impacts

- Hardlinks break, increasing data transfers.
- Kubernetes features don't add actual benefits to this stack (scheduling, scaling, etc).
- Requires 2 deployment features.
- Plex hardware encoding requires direct hw access. This can be accomplished through plugins but adds complexity.
- Prometheus will require NAS node-exporter for cross platform monitoring.

## 006 - Flux Kustomization chart/config split pattern

### Context

When migrating MetalLB to Flux, applying the Helm chart and its custom resource configuration (IPAddressPool, L2Advertisement) in a single Flux Kustomization causes a race condition on fresh clusters. The CRDs registered by the Helm chart don't exist yet when Flux tries to apply the CR instances, causing transient failures on first reconciliation.

### Decision

Split Helm chart deployments and their post-install CR configuration into two separate Flux Kustomizations with a dependsOn relationship. The chart Kustomization uses wait: true to ensure all pods are healthy before the config Kustomization applies. Adopt this as the standard pattern for any component where CRDs are installed by a Helm chart and then consumed by separate CR manifests.

### Impacts

- Clean first-apply on a fresh cluster with no transient errors or retry noise in logs.
- Explicit dependency ordering makes the reconciliation sequence readable and predictable.
- Slightly more files per component (two Flux Kustomizations + two subdirectories instead of one flat directory).
- Pattern is reusable for future components like cert-manager, Traefik, and external-dns where we want CRDs -> CRs.
- Debugging is easier since each Flux Kustomization has its own status and can fail independently.
