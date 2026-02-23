# GitLab Deployment — Method 1 (Public IP + Nginx LoadBalancer)

Domain: gitlab.kihpn.online  
GitLab Server: 192.168.1.103  
LoadBalancer: 192.168.1.101  

---

## Architecture

Internet
 → Cloudflare DNS
 → Nginx LoadBalancer
 → GitLab Server

---

## 1. Install GitLab

On GitLab Server:

```bash
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
apt update
apt install gitlab-ee=15.7.7-ee.0 -y
```

---

## 2. Generate SSL Certificate (LoadBalancer)

On LoadBalancer:

```bash
apt install certbot python3-certbot-nginx -y
```

```bash
certbot certonly \
 --standalone \
 -d gitlab.kihpn.online \
 --agree-tos -m YOUR_EMAIL
```

Certificates:

```
/etc/letsencrypt/live/gitlab.kihpn.online/
```

---

## 3. Configure Nginx Reverse Proxy

Create:

```bash
vi /etc/nginx/conf.d/gitlab.conf
```

```nginx
server {
    listen 443 ssl;
    server_name gitlab.kihpn.online;

    ssl_certificate /etc/letsencrypt/live/gitlab.kihpn.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/gitlab.kihpn.online/privkey.pem;

    location / {
        proxy_pass http://192.168.1.103;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}

server {
    listen 80;
    server_name gitlab.kihpn.online;
    return 301 https://$host$request_uri;
}
```

Restart nginx:

```bash
systemctl restart nginx
```

---

## 4. Configure GitLab External URL

On GitLab Server:

```bash
vi /etc/gitlab/gitlab.rb
```

Set:

```
external_url "https://gitlab.kihpn.online"
```

Apply config:

```bash
gitlab-ctl reconfigure
```

---

## Final Result

Cloudflare DNS
 → Nginx LoadBalancer
 → GitLab Server