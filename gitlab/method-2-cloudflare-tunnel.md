# GitLab Deployment — Method 2 (DNS-01 + Cloudflare Tunnel)

Domain: gitlab.kihpn.online  
GitLab Server: 192.168.1.103  
LoadBalancer: 192.168.1.101  

---

## Architecture

User
 → Cloudflare Edge
 → Tunnel
 → Nginx LoadBalancer
 → GitLab Server

No router port forwarding required.

---

## 1. Install GitLab

```bash
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
apt install gitlab-ee=15.7.7-ee.0 -y
```

---

## 2. Obtain SSL via Cloudflare DNS API

On LoadBalancer:

```bash
apt install certbot python3-certbot-dns-cloudflare -y
```

Create credential file:

```bash
mkdir -p ~/.secrets
vi ~/.secrets/cloudflare.ini
```

```
dns_cloudflare_api_token=YOUR_API_TOKEN
```

```bash
chmod 600 ~/.secrets/cloudflare.ini
```

Request certificate:

```bash
certbot certonly \
 --dns-cloudflare \
 --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
 -d gitlab.kihpn.online \
 --agree-tos -m YOUR_EMAIL
```

---

## 3. Configure Nginx

```nginx
server {
    listen 443 ssl;
    server_name gitlab.kihpn.online;

    ssl_certificate /etc/letsencrypt/live/gitlab.kihpn.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/gitlab.kihpn.online/privkey.pem;

    location / {
        proxy_pass http://192.168.1.103;
        proxy_set_header Host $host;
    }
}
```

```bash
systemctl restart nginx
```

---

## 4. Setup Cloudflare Tunnel

Install:

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
cloudflared tunnel create gitlab-tunnel
```

Config:

```bash
vi /etc/cloudflared/config.yml
```

```yaml
tunnel: gitlab-tunnel
credentials-file: /root/.cloudflared/UUID.json

ingress:
  - hostname: gitlab.kihpn.online
    service: https://192.168.1.101:443
  - service: http_status:404
```

Start:

```bash
cloudflared service install
systemctl start cloudflared
```

---

## 5. Configure GitLab URL

```bash
vi /etc/gitlab/gitlab.rb
```

```
external_url "https://gitlab.kihpn.online"
```

```bash
gitlab-ctl reconfigure
```

---

## Final Result

Cloudflare Tunnel
 → Nginx LoadBalancer
 → GitLab Server