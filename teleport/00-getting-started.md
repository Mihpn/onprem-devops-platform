# 🚀 Teleport Deployment — Getting Started

This guide helps you choose the correct deployment method based on your environment.

---

## 1. Choose Your Deployment Method

Before starting, check your network conditions noticed in your lab:

### Do you have a Public IP with open ports?

→ Use **Method 1 — Public IP + LoadBalancer**

---

### Does your ISP block ports (VNPT / Viettel Home Network)?

→ Use **Method 2 — Cloudflare Tunnel + DNS-01**

Recommended for most home lab environments.

---

### Need the fastest working environment?

→ Use **Method 3 — AWS Quick Deploy**

Best for testing, demos, and external access.

---

## 2. Recommended Learning Order

Follow this sequence to understand the platform architecture:

1. Read `overview.md` — high-level architecture diagram
2. Choose a deployment method:
   - `method-1-public-ip.md`
   - `method-2-cloudflare-tunnel.md`
   - `method-3-aws-quick-deploy.md`
3. Review example configs:
   - `teleport.yaml.example`
   - `teleport.service`
4. Apply supporting infrastructure:
   - `nginx/` configs
   - `cloudflared/` tunnel examples

---

## 3. Repository Philosophy

This repository is structured as an evolving platform blueprint.

Each module is added incrementally:

- Edge Layer (Cloudflare + Nginx)
- Access Layer (Teleport)
- Platform Layer (Rancher + Kubernetes)
- CI/CD Layer (GitLab)

---

> ⚠️ This is a living infrastructure lab.  
> Configuration patterns may evolve as new modules are introduced.