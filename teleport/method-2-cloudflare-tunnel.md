# Teleport On-Prem Deployment (DNS-01 + Cloudflare Tunnel + Nginx LoadBalancer)

Domain: teleport.kihpn.online  
Teleport Server: 192.168.1.102  
LoadBalancer: 192.168.1.101  

---

## Architecture

User → Cloudflare DNS → Cloudflare Tunnel → LoadBalancer (Nginx) → Teleport Server

---

## Why This Method?

Some ISPs (VNPT / Viettel) block incoming ports 80/443.

Instead of exposing a public IP:

- Request SSL using Cloudflare DNS API (DNS-01)
- Use Cloudflare Tunnel for inbound access
- Keep Nginx as internal reverse proxy

---

## 1. Obtain SSL Certificate (DNS-01)

On LoadBalancer (192.168.1.101):

```bash
apt update
apt install certbot python3-certbot-dns-cloudflare -y
```

Create credential file:

```bash
mkdir -p ~/.secrets
vi ~/.secrets/cloudflare.ini
```

Content:

```text
dns_cloudflare_api_token=YOUR_API_TOKEN
```

Set permission:

```bash
chmod 600 ~/.secrets/cloudflare.ini
```

Request certificate:

```bash
certbot certonly \
 --dns-cloudflare \
 --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
 -d teleport.kihpn.online \
 --agree-tos -m YOUR_EMAIL
```

Certificates location:

```
/etc/letsencrypt/live/teleport.kihpn.online/
```

---

## 2. Configure Nginx LoadBalancer

Create:

```bash
vi /etc/nginx/conf.d/teleport.conf
```

```nginx
server {
    listen 443 ssl;
    server_name teleport.kihpn.online;

    ssl_certificate /etc/letsencrypt/live/teleport.kihpn.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/teleport.kihpn.online/privkey.pem;

    location / {
        proxy_pass https://192.168.1.102:443;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name teleport.kihpn.online;
    return 301 https://$host$request_uri;
}
```

Reload nginx:

```bash
systemctl restart nginx
```

---

## 3. Setup Cloudflare Tunnel

Install cloudflared:

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared-linux-amd64.deb
```

Login:

```bash
cloudflared tunnel login
```

Create tunnel:

```bash
cloudflared tunnel create teleport-tunnel
```

Create config:

```bash
vi /etc/cloudflared/config.yml
```

```yaml
tunnel: teleport-tunnel
credentials-file: /root/.cloudflared/UUID.json

ingress:
  - hostname: teleport.kihpn.online
    service: https://192.168.1.101:443
  - service: http_status:404
```

Start tunnel:

```bash
cloudflared service install
systemctl start cloudflared
```

---

## 4. Configure Teleport Backend

On Teleport Server (192.168.1.102):

Ensure teleport.yaml contains:

```yaml
proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport.kihpn.online:443
```

Restart teleport:

```bash
systemctl restart teleport
```

---

## Final Result

Cloudflare Edge
 → Tunnel
 → Nginx LoadBalancer (SSL termination)
 → Teleport Server

No router port forwarding required.