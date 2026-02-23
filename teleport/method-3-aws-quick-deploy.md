\# Teleport Deployment — Method 3 (AWS Quick Deploy)



This method is designed for quick lab setup using AWS EC2.

No LoadBalancer, no Cloudflare Tunnel required.



Architecture:



User Internet

&nbsp;  ↓

AWS EC2 Public IP

&nbsp;  ↓

Teleport Server (Cloud)



Domain:

teleport.kihpn.online



---



\## Why This Method?



Use this when:



\- Local ISP blocks ports

\- Need a fast public environment

\- Want simple Teleport testing



This is the fastest working setup.



---



\## Step 1 — Create EC2 Instance



AWS Console → EC2 → Launch Instance



Recommended:



\- Ubuntu 22.04

\- t3.micro (Free Tier)

\- Allow inbound ports:



22 (SSH)

443 (HTTPS)

3025 (Teleport Auth)



Security Group Example:



Type: HTTPS

Port: 443

Source: 0.0.0.0/0



---



\## Step 2 — Connect to Server



ssh ubuntu@YOUR\_PUBLIC\_IP



Update system:



sudo apt update \&\& sudo apt upgrade -y



---



\## Step 3 — Install Teleport (Official Script)



curl https://cdn.teleport.dev/install.sh | bash -s 17.0.5



Check version:



teleport version



---



\## Step 4 — Generate Teleport Configuration



teleport configure -o file \\

&nbsp;--acme \\

&nbsp;--acme-email=YOUR\_EMAIL \\

&nbsp;--cluster-name=teleport.kihpn.online > /etc/teleport.yaml



Important:



Open config file:



sudo vi /etc/teleport.yaml



Find ACME section.



If using external SSL or Cloudflare later,

you may comment ACME lines.



---



\## Step 5 — Start Teleport



Create systemd service:



sudo tee /etc/systemd/system/teleport.service <<EOF

\[Unit]

Description=Teleport Service

After=network.target



\[Service]

ExecStart=/usr/local/bin/teleport start --config=/etc/teleport.yaml

Restart=on-failure



\[Install]

WantedBy=multi-user.target

EOF



Reload and start:



sudo systemctl daemon-reload

sudo systemctl enable teleport

sudo systemctl start teleport



Check:



sudo systemctl status teleport



---



\## Step 6 — Create Admin User



sudo tctl users add admin --roles=editor,access --logins=root,ubuntu



Teleport will generate a login URL.



Open browser and complete setup.



---



\## Step 7 — Point Domain to AWS



In Cloudflare DNS:



Create A Record:



Name: teleport

Content: YOUR\_EC2\_PUBLIC\_IP



Proxy Status: DNS Only (for ACME validation)



Wait 1–2 minutes for DNS propagation.



---



\## Step 8 — Access Teleport



Open:



https://teleport.kihpn.online



Login using created admin user.



---



\## Final Result



Public Internet

&nbsp;→ AWS EC2

&nbsp;→ Teleport Proxy + Auth



No local NAT.

No LoadBalancer needed.

Ideal for quick demos or testing.



---



\## Notes



This method is NOT intended for production HA setups.

Use LoadBalancer or Tunnel methods for hybrid deployments.



