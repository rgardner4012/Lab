# Naming & IP Allocations

## Network

| Property | Value |
| -------- | ----- |
| Subnet | 192.168.0.0/24 |
| Gateway | 192.168.0.1 |
| Domain | homelab |
| VLANs | None (flat network) |
| DNS | Pi-hole (primary: 192.168.0.245, secondary: 192.168.0.234) |

## Static IPs

### Infrastructure

| Hostname | IP | Role | Notes |
| -------- | -- | ---- | ----- |
| nas.homelab | 192.168.0.233 | UGREEN DXP6800 Pro — NAS + Docker host | Media stacks, NFS, Pi-hole secondary |
| nasbridge0.homelab | 192.168.0.235 | NAS bridge interface | Docker bridge network |
| nuc1.homelab | 192.168.0.50 | RKE2 worker (i5-8259U, 32GB) | |
| nuc2.homelab | 192.168.0.51 | RKE2 worker (i7-8559U, 32GB) | |
| nuc3.homelab | 192.168.0.52 | RKE2 worker (i7-8559U, 32GB) | |
| rancher.homelab | 192.168.0.48 | Rancher management UI | |
| nagios.homelab | 192.168.0.190 | Nagios monitoring | |

### Services

| Hostname | IP | Platform | Notes |
| -------- | -- | -------- | ----- |
| pihole.homelab | 192.168.0.234 | NAS (Docker) | Primary DNS |
| pihole2.homelab | 192.168.0.245 | K8s (MetalLB) | Secondary DNS, IP from MetalLB pool |

## MetalLB Address Pool

| Pool | Range | Mode |
| ---- | ----- | ---- |
| homelab-pool | 192.168.0.240–192.168.0.250 | L2 |

11 addresses available. Currently assigned: pihole2 (.245).

## IP Range Conventions

### This is dirty currently do to my previously existing homenetwork dhcp range (1-240)

| Range | Purpose |
| ----- | ------- |
| .1 | Gateway |
| .48–.52 | K8s / Rancher infrastructure |
| .190 | Monitoring (Nagios) |
| .233–.235 | NAS + NAS services |
| .240–.250 | MetalLB pool (K8s LoadBalancer IPs) |
