# Teleport On-Prem Deployment (Public IP + LoadBalancer)

Domain: teleport.kihpn.online  
Teleport Server: 192.168.1.102  
LoadBalancer: 192.168.1.101  

---

## 1. Install Teleport

```bash
wget https://get.gravitational.com/teleport-v13.2.0-linux-amd64-bin.tar.gz
tar -xzf teleport-v13.2.0-linux-amd64-bin.tar.gz

mv teleport/tctl /usr/local/bin/
mv teleport/tsh /usr/local/bin/
mv teleport/teleport /usr/local/bin/

teleport version
tctl version
tsh version

mkdir -p /etc/teleport
```

---

## 2. Create Teleport Configuration

Create file:

```bash
vi /etc/teleport/teleport.yaml
```

```yaml
version: v3
teleport:
  nodename: teleport
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO

auth_service:
  enabled: "yes"
  listen_addr: 0.0.0.0:3025
  cluster_name: teleport.kihpn.online
  proxy_listener_mode: multiplex

ssh_service:
  enabled: "yes"

proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport.kihpn.online:443
  https_keypairs: []
```

---

## 3. Create Systemd Service

```bash
vi /etc/systemd/system/teleport.service
```

```ini
[Unit]
Description=Teleport Service
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/teleport start --config=/etc/teleport/teleport.yaml
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Start Teleport:

```bash
systemctl daemon-reload
systemctl start teleport
systemctl status teleport
```

---

## 4. Setup SSL on LoadBalancer

```bash
apt update
apt install apache2-utils certbot python3-certbot-nginx -y
```

```bash
certbot certonly \
 --standalone \
 -d teleport.kihpn.online \
 --agree-tos \
 -m YOUR_EMAIL \
 --keep-until-expiring
```

---

## 5. Configure Nginx Reverse Proxy

```bash
vi /etc/nginx/conf.d/lb.conf
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

```bash
systemctl restart nginx
```

---

## 6. Copy SSL Certificates

```bash
scp /etc/letsencrypt/live/teleport.kihpn.online/fullchain.pem root@192.168.1.102:/etc/teleport/teleport.crt
scp /etc/letsencrypt/live/teleport.kihpn.online/privkey.pem root@192.168.1.102:/etc/teleport/teleport.key
```

---

## 7. Update Teleport Config With Certificates

```yaml
proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport.kihpn.online:443
  https_keypairs:
    - cert_file: /etc/teleport/teleport.crt
      key_file: /etc/teleport/teleport.key
```

```bash
chmod 600 /etc/teleport/teleport.key
chmod 644 /etc/teleport/teleport.crt
systemctl restart teleport
```

Create admin user:

```bash
tctl users add admin --roles=editor,access --logins=root
```

---

## 8. Optional — Deploy Teleport Cloud Node

```bash
curl https://cdn.teleport.dev/install.sh | bash -s 17.0.5
```

```bash
teleport configure -o file --acme \
 --acme-email=YOUR_EMAIL \
 --cluster-name=teleport.kihpn.online
```

```bash
systemctl enable teleport
systemctl start teleport
```

```bash
tctl users add admin --roles=editor,access --logins=root,ubuntu
```

---

## Notes

If ISP blocks ports 80/443 (VNPT/Viettel), use:

method-2-cloudflare-tunnel.md