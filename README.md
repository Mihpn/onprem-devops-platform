\# On-Prem DevOps Platform Lab



This repository documents a real on-prem DevOps platform I built and operate for learning and demos.



\## Current stack

\- Teleport (On-Prem + Cloud option)

\- GitLab (CI/CD)

\- Rancher + Kubernetes (3-node)

\- Nginx Load Balancer \& Reverse Proxy

\- Domain + Cloudflare (Tunnel \& DNS)



\## Structure

\- `architecture/` — diagrams and overview

\- `teleport/` — Teleport install \& HA / tunnel methods

\- `nginx/` — loadbalancer configs \& proxy examples

\- `cloudflared/` — sample Cloudflare Tunnel configs

\- `docs/` — operational notes and troubleshooting



\## How to use this repo

1\. Browse `teleport/` first for Teleport deployment options.

2\. Copy configs (files ending `.example`) to servers and adapt IPs/domains.

3\. Keep notes in `docs/` for any troubleshooting you encounter.



> Work in progress — I add one module at a time (commit-by-commit).

