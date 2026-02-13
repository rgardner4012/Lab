# Disaster Recovery

## Backup Checklist (To-Do)

- [ ] etcd snapshots configured (RKE2 built-in)
- [ ] Longhorn backup target configured (NAS NFS share)
- [ ] Longhorn recurring snapshots scheduled
- [X] Age private key backed up offline
- [ ] NAS app config directories backed up
- [ ] Find per app solution for NAS backed apps
  - [ ] Sonarr
  - [ ] Radarr
  - [ ] Profilarr
  - [ ] JellySeer
  - [ ] Prowlarr
  - [ ] NZBGet
  - [ ] Portainer
  - [ ] Plex
  - [ ] Rancher
- [ ] Backup restore tested

## RTO/RPO Targets

| Component | RPO | RTO | Notes |
| ----------- | ----- | ----- | ------- |
| K8s manifests | 0 (in Git) | ~30 min | Flux reconciles from Git |
| MetalLB | 0 (in Git) | ~5 min | Flux deploys Helm chart then config via depends on |
| Docker Compose stacks | 0 (in Git) | ~15 min | SSH deploy from Git |
| Longhorn volumes | Last snapshot | ~1 hr | Restore from Longhorn backup target (NFS Share) |
| OpenBao data | Last Raft snapshot | ~1 hr | Requires restore + unseal |
| Prometheus/Grafana data | Last Longhorn snapshot | ~1 hr | Dashboards are in Git (ConfigMaps) |
| Media library | N/A | N/A | Not backed up (re-downloadable, want to find a way to track what currently exists, will *arr config backups work here?) |
| App configs (Sonarr, Radarr, etc.) | Last NAS backup | ~30 min | Restore config dirs, redeploy |
| SOPS encryption key (age) | Offline backup and external Password Manager | Manual | Required to decrypt all secrets |

## RKE2 Cluster Recovery

### Total cluster loss

1. Re-provision NUCs via Elemental Operator (Rancher).
2. `flux bootstrap github` to reinstall Flux.
3. Flux reconciles all manifests from Git automatically.
4. MetalLB installs via Helm chart, then IP pool config applies once CRDs are ready.
5. Longhorn volumes are recreated empty — restore from backup target if needed.
6. LoadBalancer services get external IPs once MetalLB is healthy.

### Single node failure

RKE2 reschedules pods to remaining nodes automatically. Longhorn rebuilds volume replicas on healthy nodes (default 3 replicas).

### Longhorn volume recovery

1. If a replica is lost, Longhorn automatically rebuilds from the remaining replica(s)
2. For full volume loss, restore from Longhorn backup target (NAS NFS share)
3. Verify PVCs are bound and workloads are healthy

### MetalLB recovery

MetalLB is fully managed by Flux via the chart/config split pattern (see [ADR-006](adr.md)). On a fresh cluster:

1. `infra-metallb` Flux Kustomization installs the namespace, HelmRepository, and HelmRelease
2. Helm installs MetalLB and registers its CRDs — `wait: true` holds until pods are healthy
3. `infra-metallb-config` Flux Kustomization applies IPAddressPool and L2Advertisement
4. LoadBalancer services pick up external IPs from the 192.168.0.240-250 range

No manual intervention required.

### PiHole recovery

1. One instance runs on k8s. If it dies, it should migrate to another node. If the entire cluster is down, we still have a second instance running on
the NAS. If both are down, we've lost the network.
2. Configs need to be backed up to NAS and synced
