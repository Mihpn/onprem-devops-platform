# 🔀 Nginx — Load Balancer & Reverse Proxy

## Why This Folder Exists

This folder contains the Nginx configuration for the **central Load Balancer / Reverse Proxy** — the single entry point for all traffic into the on-premise infrastructure.

---

## Role in the Architecture

```
Cloudflare Tunnel → cloudflared → ┌─────────────────────┐
                                   │   Nginx LB Node     │
                                   │   (192.168.1.101)    │
                                   │                     │
                                   │  SSL Termination    │
                                   │  (Let's Encrypt)    │
                                   └─────┬───┬───┬───┬───┘
                                         │   │   │   │
                          ┌──────────────┘   │   │   └──────────────┐
                          ▼                  ▼   ▼                  ▼
                    Teleport          GitLab  Harbor           Rancher
                   (.102:443)       (.103:80) (.104:80)       (.108:80)
                                                    │
                                              K8s Ingress
                                             (NodePort 30080)
                                                    │
                                              ShopNow App
```

**The Nginx LB node performs 3 critical functions:**

1. **SSL Termination** — Holds all Let's Encrypt certificates (obtained via DNS-01 challenge through Cloudflare API). Backend services receive plain HTTP, saving CPU resources.
2. **Reverse Proxy** — Routes traffic based on `server_name` to the correct internal backend server.
3. **Tunnel Endpoint** — The `cloudflared` daemon also runs on this node, sending traffic from Cloudflare into Nginx.

---

## Configuration Example

See [`lb.conf.example`](lb.conf.example) — a sample virtual host configuration for Teleport.

**Pattern used for all services:**

```nginx
# HTTPS — proxy to backend
server {
    listen 443 ssl;
    server_name <service>.kihpn.online;

    ssl_certificate /etc/letsencrypt/live/<service>.kihpn.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<service>.kihpn.online/privkey.pem;

    location / {
        proxy_pass http://<BACKEND_IP>:<PORT>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name <service>.kihpn.online;
    return 301 https://$host$request_uri;
}
```

## Virtual Hosts Configured

| Domain | Backend Target | Service |
|---|---|---|
| `teleport.kihpn.online` | `192.168.1.102:443` | Teleport (WebSocket upgrade enabled) |
| `gitlab.kihpn.online` | `192.168.1.103:80` | GitLab |
| `harbor.kihpn.online` | `192.168.1.104:80` | Harbor Registry |
| `rancher.kihpn.online` | `192.168.1.108:80` | Rancher |
| `shopnow.kihpn.online` | `K8s NodePort :30080` | ShopNow App (via Ingress) |

---

## Why Nginx Instead of HAProxy or Cloud LB?

- **Familiarity** — Nginx is the most widely used reverse proxy in enterprise environments
- **Flexibility** — Handles both L4 (TCP stream) and L7 (HTTP) proxying
- **SSL Termination** — Offloads TLS processing from backend services
- **On-Prem** — No dependency on cloud providers; full control over routing rules
