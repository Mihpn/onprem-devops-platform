# ☁️ Cloudflare Tunnel (`cloudflared`)

## Why This Folder Exists

This folder contains the configuration for **Cloudflare Tunnel** — the critical component that makes the entire on-premise platform accessible from the Internet **without opening any inbound ports** on the home network router.

---

## The Problem It Solves

Vietnamese ISPs (VNPT, Viettel) **block inbound traffic on ports 80 and 443** for residential connections. This means:

- ❌ Cannot point a domain directly to the home IP
- ❌ Cannot use Let's Encrypt HTTP-01 challenge (needs port 80)
- ❌ Cannot expose any service (GitLab, Harbor, Rancher) to the Internet

## The Solution

**Cloudflare Tunnel** creates a **secure, outbound-only** encrypted connection from the on-prem LB node to Cloudflare's global edge network. Traffic flow:

```
User → Cloudflare Edge → Tunnel (outbound from LB) → Nginx LB → Backend Services
```

- No port forwarding needed on the router
- No public IP required
- All traffic is encrypted and passes through Cloudflare's DDoS protection

---

## How It Fits in the Architecture

```
Internet
  → Cloudflare DNS (*.kihpn.online)
    → Cloudflare Tunnel
      → cloudflared daemon (LB Node)
        → Nginx Reverse Proxy (SSL termination)
          → GitLab / Harbor / Rancher / Teleport / K8s Ingress
```

The `cloudflared` daemon runs as a **systemd service** on the Load Balancer node and maintains a persistent outbound connection to Cloudflare. When a user visits `gitlab.kihpn.online`, Cloudflare routes the request through this tunnel to the local Nginx.

---

## Configuration

See [`config.yml.example`](config.yml.example) for the tunnel routing configuration.

**Key fields:**

| Field | Purpose |
|---|---|
| `tunnel` | Name of the Cloudflare Tunnel |
| `credentials-file` | Authentication JSON from `cloudflared tunnel login` |
| `ingress[].hostname` | Domain to route (e.g., `teleport.kihpn.online`) |
| `ingress[].service` | Backend target (e.g., `https://192.168.1.101:443`) |

---

## Quick Setup Commands

```bash
# Install
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Authenticate
sudo cloudflared tunnel login

# Create tunnel
sudo cloudflared tunnel create <TUNNEL_NAME>

# Route DNS
sudo cloudflared tunnel route dns <TUNNEL_NAME> <HOSTNAME>

# Install as service
sudo cloudflared service install
sudo systemctl enable --now cloudflared
```
