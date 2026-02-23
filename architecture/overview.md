\# Platform Architecture Overview



Internet

 -> Domain

 -> LoadBalancer

 -> Nginx Reverse Proxy

 -> Rancher

 -> Kubernetes Cluster

 -> CI/CD via GitLab





Internet

&nbsp; ↓

Domain (Cloudflare)

&nbsp; ↙        ↘

Cloudflare CDN  Cloudflare Tunnel (optional)

&nbsp; ↓                ↓

Public LoadBalancer (nginx) — reverse proxy → Teleport (192.168.100.102)

&nbsp;                               → Other internal services



Notes:

\- If you have a public IP, use certbot on LB and proxy to internal Teleport.

\- If ISP blocks ports / no public IP, use Cloudflare Tunnel to expose Teleport securely.

