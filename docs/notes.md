\# Operational Notes



\## ISP Issues



VNPT / Viettel may block inbound ports.

Use Cloudflare Tunnel as alternative.



\## Lessons Learned



\- DNS-01 works without opening ports

\- Nginx should terminate SSL before teleport

\- Tunnel should point to LoadBalancer instead of backend

