

# On-Prem DevOps Platform Lab

This repository documents an on-prem DevOps platform I run for learning and demos.

## Current stack
- Teleport (On-Prem + Cloud option)
- GitLab (CI/CD)
- Rancher + Kubernetes (3-node)
- Nginx Load Balancer & Reverse Proxy
- Domain + Cloudflare (Tunnel & DNS)

## Structure
- `architecture/` — diagrams and overview (Mermaid)
- `teleport/` — Teleport install & tunnel methods
- `nginx/` — loadbalancer configs & proxy examples
- `cloudflared/` — sample Cloudflare Tunnel configs
- `docs/` — operational notes and troubleshooting

## How to use this repo
1. Read `architecture/overview.md` for the high-level diagram.
2. Choose a deployment method in `teleport/` (method-1 / 2 / 3).
3. Copy `*.example` files to servers, edit IPs/domains, then apply.
4. Find nginx configs in `nginx/` and Cloudflare samples in `cloudflared/`.

## Contributing
- Use branches per feature: `git checkout -b docs/architecture-update`
- Open PR when ready.

_Work in progress — add one module at a time (commit-by-commit)._
