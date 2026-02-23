# 🐳 Harbor Registry: Zero-Trust Architecture via Cloudflare Tunnel & Nginx

This repository provides a comprehensive, production-grade guide to deploying an on-premise **Harbor Image Registry** shielded by a **Cloudflare Zero Trust Tunnel** with **Nginx** acting as the Reverse Proxy and handling SSL Termination.

This architecture enforces strict zero-trust principles (no exposed inbound ports) while optimizing the Harbor backend's resource utilization by delegating SSL/TLS encryption overhead to the edge/load balancer.

---

## 🏗️ Architecture Topology

```text
[Clients / K8s Cluster] 
       │
       ▼ (HTTPS / Port 443)
[Cloudflare Edge Network] 
       │
       ▼ (Secure Outbound Tunnel)
[Cloudflared Daemon] ──(No TLS Verify)──┐
(Node 1: <LB_IP>)                       │
                                        ▼ (HTTPS / Port 443)
                             [Nginx Reverse Proxy] ─(SSL Termination)─┐
                             (Node 1: <LB_IP>)                        │
                                                                      ▼ (HTTP / Port 80)
                                                               [Harbor Registry Backend]
                                                               (Node 2: <HARBOR_IP>)
```

## 📋 Prerequisites

- **Domain:** A domain actively managed on Cloudflare (e.g., `harbor.yourdomain.com`).
- **Load Balancer Node (`<LB_IP>`):** A Linux server pre-provisioned with Nginx, Certbot (with Cloudflare DNS plugin), and `cloudflared`.
- **Harbor Backend Node (`<HARBOR_IP>`):** A Linux server pre-provisioned with Docker and Docker Compose.

---

## 🚀 Deployment Guide

### Phase 1: Cloudflare Tunnel Configuration (Zero Trust Dashboard)

To establish the secure ingress routing without exposing local ports:

1. Navigate to the [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/).
2. Go to **Networks** > **Connectors** (Tunnels) > Select your target tunnel > **Configure**.
3. Under the **Public Hostname** tab, add the following route:
   - **Public Hostname:** `harbor.yourdomain.com`
   - **Service:** `HTTPS` -> `<LB_IP>:443`
4. ⚠️ **Critical TLS Configuration:** Expand **Additional application settings**, navigate to the **TLS** tab, and toggle **No TLS Verify** to `ON`. This