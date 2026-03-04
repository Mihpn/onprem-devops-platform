# 🌐 Networking — Platform Network Design

## Why This Folder Exists

This folder documents the **network architecture** of the on-premise platform — how VMs communicate internally, how external traffic reaches services, and the network segmentation decisions made.

---

## Network Topology

```
                    ┌─────────────────────────────────────────────┐
                    │        Home Network (NAT behind ISP)        │
                    │                                             │
 ┌─────────────────────────────────────────────────────────────┐  │
 │  Infrastructure VLAN — 192.168.1.0/24                       │  │
 │                                                             │  │
 │  .101  Nginx LB + cloudflared   (Edge / Entry Point)        │  │
 │  .102  Teleport On-Prem         (Access Control)            │  │
 │  .103  GitLab Server            (Source Code + CI)          │  │
 │  .104  Harbor Registry          (Private Images)            │  │
 │  .105  K8s Node 1               (Control Plane)             │  │
 │  .106  K8s Node 2               (Control Plane)             │  │
 │  .107  K8s Node 3               (Control Plane)             │  │
 │  .108  Rancher                  (Cluster Management)        │  │
 │                                                             │  │
 └─────────────────────────────────────────────────────────────┘  │
                    │                                             │
                    │  Outbound ONLY → Cloudflare Tunnel          │
                    │  (No inbound ports opened on router)        │
                    └─────────────────────────────────────────────┘
```

---

## Key Design Decisions

### 1. Single Subnet (Flat Network)

All VMs reside on `192.168.1.0/24`. In a production enterprise, you would segment:
- DMZ for LB
- Management VLAN for Teleport/Rancher
- Application VLAN for K8s nodes

For a home lab, a flat network simplifies management while still demonstrating the architecture.

### 2. Zero Inbound Ports

The router has **no port forwarding rules**. All external access flows through:
```
Internet → Cloudflare Edge → Tunnel (outbound from .101) → Nginx LB → Backend
```

### 3. Internal Communication

- **VM-to-VM** — Direct communication via internal IPs (e.g., Nginx → GitLab at `192.168.1.103:80`)
- **K8s Pod-to-Pod** — Calico CNI handles pod networking with overlay
- **K8s Service Discovery** — CoreDNS resolves service names within the cluster
- **K8s Ingress** — NGINX Ingress Controller exposes services via NodePort (30080/30443)

### 4. DNS Resolution

| Source | Domain | Resolution |
|---|---|---|
| **External** | `*.kihpn.online` | Cloudflare DNS → Tunnel CNAME |
| **Internal** | `*.kihpn.online` | `/etc/hosts` or local DNS → LB at 192.168.1.101 |
| **K8s Internal** | `<svc>.<ns>.svc.cluster.local` | CoreDNS |
