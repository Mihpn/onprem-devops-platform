# Rancher — Method A (Public IP + Nginx LoadBalancer + Let's Encrypt)

**When to use:** you have a public IP or can port-forward 80/443 to an Nginx LB.  
**Goal:** LB terminates TLS (Let's Encrypt) and reverse-proxies to Rancher running in Docker Compose.

---

## Quick topology
```
Internet -> Cloudflare DNS -> Public IP (Nginx LB) -> Rancher Host (Docker Compose)
```

---

## Prerequisites
- Domain `rancher.kihpn.online` (DNS record points to LB public IP)
- LB host (Ubuntu) with `nginx` & `certbot`
- Rancher host with Docker & Docker Compose
- Basic SSH access between machines

---

## 1) Deploy Rancher (Docker Compose) — Rancher host (e.g. 192.168.1.108)

Create `docker-compose.yml` (repo: `rancher-k8s/docker-compose.yml`):

```yaml
version: '3'
services:
  rancher-server:
    image: rancher/rancher:v2.9.2
    container_name: rancher-server
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./data/:/var/lib/rancher
    command:
      - --no-cacerts
    environment:
      - CATTLE_AGENT_TLS_MODE=system-store
    privileged: true
```

Start:

```bash
cd /path/to/rancher-k8s
docker compose up -d
```

> Note: `--no-cacerts` is used when TLS is managed externally (LB or Cloudflare). Remove it if you want Rancher to manage certs.

---

## 2) Obtain TLS on LB (Let’s Encrypt) — LB host

Install and request:

```bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
sudo certbot certonly --nginx -d rancher.kihpn.online -m YOUR_EMAIL --agree-tos --non-interactive
```

Certificates will be in:

```
/etc/letsencrypt/live/rancher.kihpn.online/
```

---

## 3) Nginx config on LB — `/etc/nginx/conf.d/rancher.conf`

```nginx
server {
    listen 443 ssl;
    server_name rancher.kihpn.online;

    ssl_certificate /etc/letsencrypt/live/rancher.kihpn.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rancher.kihpn.online/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
        proxy_pass http://192.168.1.108:80;   # replace with Rancher host IP
        proxy_read_timeout 300;
    }
}

server {
    listen 80;
    server_name rancher.kihpn.online;
    return 301 https://$host$request_uri;
}
```

Reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 4) Verify

- Visit `https://rancher.kihpn.online` externally.
- Get bootstrap password:

```bash
docker logs rancher-server 2>&1 | grep "Bootstrap Password:"
```

- Check container:

```bash
docker ps | grep rancher
docker logs rancher-server -n 200
```

---

## Troubleshooting

- `502` from nginx: `curl -vk http://192.168.1.108:80` from LB; check Rancher container logs.
- Certbot errors: ensure port 80/443 reachable and DNS points to LB.
- If LetsEncrypt fails, use DNS-01 or Cloudflare method.

---

## Quick checklist

- [ ] Rancher container running
- [ ] LB certificate issued
- [ ] Nginx config proxies to Rancher host
- [ ] External access works, bootstrap password retrieved

---

## Notes

- Docker Compose is for labs. For production use HA RKE2 or managed installations.
- Do not commit certs or secrets to Git.