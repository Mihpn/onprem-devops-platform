# 📝 Operational Notes & Troubleshooting

## Why This Folder Exists

A running on-premise platform generates real operational knowledge — edge cases, ISP quirks, and lessons that can't be found in any documentation. This file captures those insights.

---

## ISP Issues (Vietnam)

| ISP | Issue | Impact |
|---|---|---|
| **VNPT** | Blocks inbound ports 80, 443 | Cannot use HTTP-01 SSL challenge, no direct domain access |
| **Viettel** | Same port blocking behavior | Same workaround required |

**Workaround:** Cloudflare Tunnel (outbound-only connection) + DNS-01 challenge for SSL certificates.

---

## Lessons Learned

### 1. DNS-01 Challenge Works Without Opening Ports
Let's Encrypt can verify domain ownership via a DNS TXT record (through the Cloudflare API) instead of serving a file on port 80. This is the **only viable option** when ISP blocks ports.

```bash
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
  -d <domain>
```

### 2. Nginx Should Terminate SSL Before Teleport
Teleport has its own TLS handling, but when placed behind an Nginx LB:
- Let Nginx handle SSL termination (using Let's Encrypt certs)
- Proxy to Teleport's HTTPS port with `proxy_pass https://...`
- Enable WebSocket upgrade headers (required for Teleport sessions)

### 3. Tunnel Should Point to LoadBalancer, Not Directly to Backend
**Wrong:** `cloudflared → GitLab backend directly`  
**Right:** `cloudflared → Nginx LB → GitLab backend`

**Why:** Nginx handles SSL, logging, rate limiting, and header injection. Bypassing it loses all observability and security controls.

### 4. Harbor "No TLS Verify" in Cloudflare Tunnel
When tunneling to Harbor through Nginx (with self-signed or Let's Encrypt certs):
- In Cloudflare Zero Trust Dashboard → Tunnel Config → **Enable "No TLS Verify"**
- This tells the Cloudflare Tunnel to trust the Nginx SSL cert without CA validation

### 5. K8s Pod Eviction Due to Resource Pressure
**Symptom:** Pods getting `Evicted` status with `DiskPressure` or `MemoryPressure`.  
**Fix:** Set strict resource `requests` and `limits` on every pod + HPA with conservative scaling (max 2 replicas, 50% CPU target).
