# Rancher — Method B (Cloudflare DNS-01 + Cloudflare Tunnel -> LB or Direct)

**When to use:** ISP blocks inbound ports or you prefer Cloudflare to terminate TLS.  
**Goal:** Cloudflare Edge TLS → cloudflared Tunnel → LB or direct Rancher host (HTTP). LB may still do proxying for other services.

---

## Quick topology
```
Internet -> Cloudflare (DNS & Tunnel) -> cloudflared -> LB (Nginx) -> Rancher (HTTP)
```

---

## Prerequisites
- Domain `rancher.kihpn.online` in Cloudflare
- Cloudflare API token (Zone:DNS Edit) if using DNS-01 certs
- Host to run cloudflared (can be the LB or any outbound-capable host)
- Rancher host with Docker & Docker Compose

---

## 1) Deploy Rancher (same docker-compose as Method A)

Use the same `docker-compose.yml` and start:

```bash
docker compose up -d
```

Rancher listens on port 80 by default (we will rely on Cloudflare for TLS).

---

## 2) (Optional) Obtain Let's Encrypt cert via DNS-01 on LB

If you want certs on LB (not required if Cloudflare terminates TLS):

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-dns-cloudflare
mkdir -p ~/.secrets
cat > ~/.secrets/cloudflare.ini <<'EOF'
dns_cloudflare_api_token = YOUR_CF_API_TOKEN
EOF
chmod 600 ~/.secrets/cloudflare.ini

sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
  -d rancher.kihpn.online \
  --agree-tos -m YOUR_EMAIL --non-interactive
```

---

## 3) Install & configure cloudflared (on LB or remote host)

Install:

```bash
wget -qO cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb || sudo apt --fix-broken install -y && sudo dpkg -i cloudflared.deb
```

Login (interactive):

```bash
sudo cloudflared login
```

Create tunnel:

```bash
sudo cloudflared tunnel create rancher-tunnel
# note printed TUNNEL-UUID (replace below)
```

Route DNS:

```bash
sudo cloudflared tunnel route dns rancher-tunnel rancher.kihpn.online
```

Create `/etc/cloudflared/config.yml`:

```yaml
tunnel: <TUNNEL-UUID>
credentials-file: /root/.cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: rancher.kihpn.online
    service: http://192.168.1.108:80   # direct to Rancher backend
  - service: http_status:404
```

Install service and start:

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
```

---

## 4) (Optional) LB Nginx config when tunneling to LB

If you route to LB (cloudflared -> LB), LB can proxy to Rancher or other services. Example LB proxy to Rancher (HTTP):

```nginx
server {
    listen 443 ssl;
    server_name rancher.kihpn.online;

    ssl_certificate /etc/letsencrypt/live/rancher.kihpn.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rancher.kihpn.online/privkey.pem;

    location / {
        proxy_pass http://192.168.1.108:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Note: If Cloudflare handles TLS, you can also accept plain HTTP on LB.

---

## 5) Verify

- From outside: `curl -vk https://rancher.kihpn.online/` (should get Cloudflare TLS and Rancher page)
- Get bootstrap password:

```bash
docker logs rancher-server 2>&1 | grep "Bootstrap Password:"
```

- Check cloudflared logs:

```bash
sudo journalctl -u cloudflared -f
```

---

## Troubleshooting

- DNS not resolving: `dig +short rancher.kihpn.online` — should show Cloudflare tunnel record.
- No response: check `sudo journalctl -u cloudflared`, `curl -vk http://192.168.1.108:80` from the tunnel host.
- If UI shows mixed content issues, ensure Cloudflare is terminating TLS and backend is HTTP (or configure LB with certs).

---

## Security notes

- Do not commit `~/.secrets/cloudflare.ini` or certificates to Git.
- Use Cloudflare Access policies if exposing admin UI widely.
- Backup `/var/lib/rancher` regularly.

---

## Quick checklist

- [ ] Rancher container up
- [ ] cloudflared tunnel created and running
- [ ] DNS routed to tunnel (Cloudflare)
- [ ] External access verified
- [ ] Bootstrap password retrieved
