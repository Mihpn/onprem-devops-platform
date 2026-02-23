\# Teleport On-Prem Deployment (DNS-01 + Cloudflare Tunnel + Nginx LoadBalancer)



This guide describes a hybrid deployment method designed for environments

where ISP blocks incoming ports (VNPT/Viettel home networks).



Architecture:



User → Cloudflare DNS → Cloudflare Tunnel → LoadBalancer (Nginx) → Teleport Server



Domain:

teleport.kihpn.online



Teleport Server:

192.168.1.102



LoadBalancer:

192.168.1.101



---



\## Why This Method?



Some ISPs block ports 80/443 or do not allow router configuration.

Instead of exposing a public IP, we:



\- Request SSL certificate using Cloudflare DNS API (DNS-01 challenge)

\- Use Cloudflare Tunnel for secure inbound access

\- Keep Nginx as reverse proxy internally



---



\## Step 1 — Obtain SSL using Cloudflare DNS API



On LoadBalancer (192.168.1.101):



Install packages:



apt update

apt install certbot python3-certbot-dns-cloudflare -y



Create credential file:



mkdir -p ~/.secrets

vi ~/.secrets/cloudflare.ini



Content:



dns\_cloudflare\_api\_token=YOUR\_API\_TOKEN



Set permission:



chmod 600 ~/.secrets/cloudflare.ini



Request certificate:



certbot certonly \\

&nbsp;--dns-cloudflare \\

&nbsp;--dns-cloudflare-credentials ~/.secrets/cloudflare.ini \\

&nbsp;-d teleport.kihpn.online \\

&nbsp;--agree-tos -m YOUR\_EMAIL



Certificates will be stored at:



/etc/letsencrypt/live/teleport.kihpn.online/



---



\## Step 2 — Configure Nginx LoadBalancer



Create:



/etc/nginx/conf.d/teleport.conf



server {

&nbsp;   listen 443 ssl;

&nbsp;   server\_name teleport.kihpn.online;



&nbsp;   ssl\_certificate /etc/letsencrypt/live/teleport.kihpn.online/fullchain.pem;

&nbsp;   ssl\_certificate\_key /etc/letsencrypt/live/teleport.kihpn.online/privkey.pem;



&nbsp;   location / {

&nbsp;       proxy\_pass https://192.168.1.102:443;

&nbsp;       proxy\_http\_version 1.1;

&nbsp;       proxy\_set\_header Host $host;

&nbsp;       proxy\_set\_header Upgrade $http\_upgrade;

&nbsp;       proxy\_set\_header Connection "upgrade";

&nbsp;       proxy\_set\_header X-Real-IP $remote\_addr;

&nbsp;   }

}



Restart nginx:



systemctl restart nginx



---



\## Step 3 — Setup Cloudflare Tunnel



Install cloudflared:



wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

dpkg -i cloudflared-linux-amd64.deb



Login:



cloudflared tunnel login



Create tunnel:



cloudflared tunnel create teleport-tunnel



Create config:



/etc/cloudflared/config.yml



tunnel: teleport-tunnel

credentials-file: /root/.cloudflared/UUID.json



ingress:

&nbsp; - hostname: teleport.kihpn.online

&nbsp;   service: https://192.168.1.101:443

&nbsp; - service: http\_status:404



Start tunnel:



cloudflared service install

systemctl start cloudflared



---



\## Step 4 — Configure Teleport Backend



On Teleport Server (192.168.1.102):



Ensure teleport.yaml contains:



proxy\_service:

&nbsp; enabled: "yes"

&nbsp; web\_listen\_addr: 0.0.0.0:443

&nbsp; public\_addr: teleport.kihpn.online:443



Restart:



systemctl restart teleport



---



\## Final Result



User traffic flow:



Cloudflare Edge

&nbsp;→ Tunnel

&nbsp;→ Nginx LoadBalancer (SSL termination)

&nbsp;→ Teleport Server



No router port forwarding required.

