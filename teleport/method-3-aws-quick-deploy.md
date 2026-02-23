# Teleport Deployment — Method 3 (AWS Quick Deploy)

Domain: teleport.kihpn.online  
Deployment Type: Cloud (AWS EC2)  
Topology: Direct Public Access  

---

## Architecture

User Internet
 → AWS EC2 Public IP
 → Teleport Server (Proxy + Auth)

This method does NOT use:

- Local LoadBalancer
- Cloudflare Tunnel

It is designed for quick testing or external lab access.

---

## Why This Method?

Use this deployment when:

- Local ISP blocks ports
- Need a fast public Teleport environment
- Want a simple standalone setup

This is the fastest working Teleport deployment.

---

## 1. Create EC2 Instance

AWS Console → EC2 → Launch Instance

Recommended configuration:

- OS: Ubuntu 22.04
- Instance type: t3.micro (Free Tier)
- Storage: Default

Security Group inbound rules:

- TCP 22 → SSH
- TCP 443 → Teleport Web UI
- TCP 3025 → Teleport Auth

Example:

| Type | Port | Source |
|------|------|--------|
| SSH | 22 | Your IP |
| HTTPS | 443 | 0.0.0.0/0 |
| Custom TCP | 3025 | 0.0.0.0/0 |

---

## 2. Connect to Server

```bash
ssh ubuntu@YOUR_PUBLIC_IP
```

Update system:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Install Teleport (Official Script)

```bash
curl https://cdn.teleport.dev/install.sh | bash -s 17.0.5
```

Verify installation:

```bash
teleport version
```

---

## 4. Generate Teleport Configuration

```bash
teleport configure -o file \
 --acme \
 --acme-email=YOUR_EMAIL \
 --cluster-name=teleport.kihpn.online \
 > /etc/teleport.yaml
```

Open and review config:

```bash
sudo vi /etc/teleport.yaml
```

Notes:

- ACME will automatically request SSL from Let's Encrypt.
- If later using Cloudflare or external SSL, you may comment the ACME section.

---

## 5. Create Systemd Service

```bash
sudo tee /etc/systemd/system/teleport.service <<EOF
[Unit]
Description=Teleport Service
After=network.target

[Service]
ExecStart=/usr/local/bin/teleport start --config=/etc/teleport.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

Reload and start service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable teleport
sudo systemctl start teleport
sudo systemctl status teleport
```

---

## 6. Create Admin User

```bash
sudo tctl users add admin --roles=editor,access --logins=root,ubuntu
```

Teleport will generate a login URL.

Open it in browser and complete setup.

---

## 7. Point Domain to AWS

In Cloudflare DNS:

Create A Record:

Name: teleport  
Content: YOUR_EC2_PUBLIC_IP  

Set Proxy Status:

DNS Only (required for ACME validation)

Wait 1–2 minutes for DNS propagation.

---

## 8. Access Teleport

Open:

```
https://teleport.kihpn.online
```

Login using the admin account created earlier.

---

## Final Result

Public Internet
 → AWS EC2
 → Teleport Proxy + Auth

No local NAT.
No LoadBalancer required.
Ideal for quick testing environments.

---

## Notes

This deployment is NOT recommended for production HA setups.

For hybrid or on-prem environments use:

- Method 1 — Public IP + LoadBalancer
- Method 2 — DNS-01 + Cloudflare Tunnel