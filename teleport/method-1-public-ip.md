\# Teleport On-Prem Deployment (Public IP + Load Balancer)



This guide describes a full on-premise Teleport deployment using:



\- Domain: teleport.kihpn.online

\- Teleport Server: 192.168.1.102

\- Load Balancer (Nginx): 192.168.1.101



Architecture Flow:



Internet

&nbsp;→ teleport.kihpn.online

&nbsp;→ Nginx Load Balancer (SSL Termination)

&nbsp;→ Teleport Server (192.168.1.102)



---



\## 1. Install Teleport on Teleport Server (192.168.1.102)



Install an older version first (for upgrade testing).



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



\## 2. Create Teleport Configuration

Create:

vi /etc/teleport/teleport.yaml



\## config file.yaml



version: v3

teleport:

&nbsp; nodename: teleport

&nbsp; data\_dir: /var/lib/teleport

&nbsp; log:

&nbsp;   output: stderr

&nbsp;   severity: INFO

&nbsp;   format:

&nbsp;     output: text



auth\_service:

&nbsp; enabled: "yes"

&nbsp; listen\_addr: 0.0.0.0:3025

&nbsp; cluster\_name: teleport.kihpn.online

&nbsp; proxy\_listener\_mode: multiplex



ssh\_service:

&nbsp; enabled: "yes"



proxy\_service:

&nbsp; enabled: "yes"

&nbsp; web\_listen\_addr: 0.0.0.0:443

&nbsp; public\_addr: teleport.kihpn.online:443

&nbsp; https\_keypairs: \[]

&nbsp; https\_keypairs\_reload\_interval: 0s

--------------------------------------



\## 3. Create Systemd Service

Creat:

vi /etc/systemd/system/teleport.service



\[Unit]

Description=Teleport Service

Documentation=https://gravitational.com/teleport/docs

After=network.target



\[Service]

User=root

ExecStart=/usr/local/bin/teleport start --config=/etc/teleport/teleport.yaml

Restart=on-failure

LimitNOFILE=65536



\[Install]

WantedBy=multi-user.target

--------------------------------------

Start Teleport:

systemctl daemon-reload

systemctl start teleport

systemctl status teleport



--------------------------------------



\## 4. Setup SSL on Load Balancer (192.168.1.101)



Install required packages:



apt update

apt install apache2-utils certbot python3-certbot-nginx -y



Request certificate:



certbot certonly \\

&nbsp;--standalone \\

&nbsp;-d teleport.kihpn.online \\

&nbsp;--agree-tos \\

&nbsp;-m YOUR\_EMAIL \\

&nbsp;--keep-until-expiring

--------------------------------------



\## 5. Configure Nginx Reverse Proxy



\#Create:



/etc/nginx/conf.d/lb.conf

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

&nbsp;       proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;

&nbsp;       proxy\_set\_header X-Forwarded-Proto $scheme;

&nbsp;   }

}



server {

&nbsp;   listen 80;

&nbsp;   server\_name teleport.kihpn.online;

&nbsp;   return 301 https://$host$request\_uri;

}



\#Reload nginx:



systemctl restart nginx

--------------------------------------------



\##6. Copy SSL Certificates to Teleport Server



From Load Balancer:



scp /etc/letsencrypt/live/teleport.kihpn.online/fullchain.pem root@192.168.1.102:/etc/teleport/teleport.crt

scp /etc/letsencrypt/live/teleport.kihpn.online/privkey.pem root@192.168.1.102:/etc/teleport/teleport.key



\##7. Update Teleport Config With Certificates



\#Edit:



/etc/teleport/teleport.yaml



Update proxy section:



proxy\_service:

&nbsp; enabled: "yes"

&nbsp; web\_listen\_addr: 0.0.0.0:443

&nbsp; public\_addr: teleport.kihpn.online:443

&nbsp; https\_keypairs:

&nbsp;   - cert\_file: /etc/teleport/teleport.crt

&nbsp;     key\_file: /etc/teleport/teleport.key



Set permissions:



chmod 600 /etc/teleport/teleport.key

chmod 644 /etc/teleport/teleport.crt



Restart Teleport:



systemctl restart teleport



Create admin user:



tctl users add admin --roles=editor,access --logins=root



-------------------------------------------------------

\##8. Optional: Deploy Teleport Cloud Node (VPS)



Install Teleport:



curl https://cdn.teleport.dev/install.sh | bash -s 17.0.5



Generate config:



teleport configure -o file --acme \\

&nbsp;--acme-email=YOUR\_EMAIL \\

&nbsp;--cluster-name=teleport.kihpn.online



Comment out ACME section if SSL is handled by Load Balancer.



Start service:



systemctl enable teleport

systemctl start teleport



Create admin user:



tctl users add admin --roles=editor,access --logins=root,ubuntu

Notes



If you cannot open ports 80/443 due to ISP restrictions (VNPT/Viettel),

use the Cloudflare Tunnel method instead (see method-2-cloudflare-tunnel.md).



