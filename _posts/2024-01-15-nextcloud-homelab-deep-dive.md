---
title: "Building a Bulletproof Nextcloud Homelab: Lessons from the Trenches"
date: 2024-01-15
tags: [homelab, selfhosted, nextcloud, docker, traefik, troubleshooting]
layout: post
---

# 📘 Overview
> I set out to build a highly private self-hosted file sync service using Nextcloud in a segmented homelab, routed via Cloudflare Tunnel — but ran into severe issues with NAT reflection, authentication layers, and reverse proxy SSL config. Here's everything I learned from 3 weeks of troubleshooting.

What started as a weekend project to replace Google Drive turned into a masterclass in networking, containerization, and the subtle art of reading documentation more carefully. This post chronicles the journey from "this should be easy" to "why is everything on fire" to finally achieving a rock-solid, secure Nextcloud deployment.

---

# 🧱 Project / Setup Context

- **Stack / Tools Used**:
  - Proxmox VE 8.1 (hypervisor)
  - pfSense 2.7 (firewall/router)
  - Docker + Docker Compose
  - Traefik v3 (reverse proxy)
  - Nextcloud 28 (AIO image)
  - Cloudflare Tunnel
  - PostgreSQL 15
  - Redis 7

- **Initial Goals**:
  - Replace Google Drive with self-hosted solution
  - Maintain security with network segmentation
  - Enable external access without port forwarding
  - Achieve 99.9% uptime for family use
  - Support mobile apps and desktop sync

- **Constraints**:
  - Residential internet (no static IP)
  - Limited to 32GB RAM across all VMs
  - Must work behind CGNAT
  - Wife Acceptance Factor (WAF) = critical

---

# 💥 Roadblocks & Shortcomings

## 🚨 Problem #1: Traefik Can't Reach Nextcloud Container
- **What happened**: Traefik was throwing `502 Bad Gateway` errors constantly, even though Nextcloud appeared healthy
- **Symptoms**: 
  - `docker logs traefik` showed connection refused errors
  - Nextcloud container logs were clean
  - Direct container access via IP worked fine
- **Why it was frustrating**: The exact same setup worked in my test environment, but failed in production with seemingly identical configs

## 🧩 Problem #2: Cloudflare Tunnel SSL Termination Chaos
- **What happened**: Mixed SSL termination between Cloudflare and Traefik caused certificate validation loops
- **Symptoms**:
  - Nextcloud web interface loaded but showed "untrusted domain" warnings
  - Mobile apps couldn't connect at all
  - Browser showed mixed content warnings
- **Why it was confusing**: Cloudflare's documentation suggested their setup should "just work" with any backend

## 🔥 Problem #3: Database Connection Pool Exhaustion
- **What happened**: After 2-3 days of uptime, Nextcloud would become unresponsive
- **Symptoms**:
  - PostgreSQL logs showed "too many connections" errors
  - Nextcloud couldn't perform any database operations
  - Required container restart to resolve
- **Why it was maddening**: This only happened in production with real usage patterns, never during testing

## 🌐 Problem #4: NAT Reflection Breaking Internal Access
- **What happened**: External domain worked from outside network but not from internal clients
- **Symptoms**:
  - Nextcloud accessible via `nextcloud.example.com` from mobile data
  - Same URL timed out from home WiFi
  - Internal IP access worked but broke mobile app sync
- **Why it was painful**: Family members couldn't sync files when at home, defeating the whole purpose

---

# 🧠 How I Resolved the Issues

## 🔧 Fix for Problem #1: Docker Network Architecture Overhaul
**What didn't work:**
- Default bridge networking
- Host networking (security concerns)
- Multiple attempts at port mapping

**What ultimately worked:**
```yaml
# docker-compose.yml
networks:
  traefik:
    external: true
  nextcloud:
    internal: true

services:
  nextcloud:
    networks:
      - traefik
      - nextcloud
    labels:
      - "traefik.docker.network=traefik"
```

**Key insight**: Traefik needs explicit network configuration when containers use multiple networks. The `traefik.docker.network` label was the missing piece.

## 🛠️ Fix for Problem #2: SSL Termination Strategy
**The solution**: Terminate SSL at Cloudflare only, use HTTP internally
```yaml
# Traefik dynamic config
http:
  routers:
    nextcloud:
      rule: "Host(`nextcloud.example.com`)"
      service: nextcloud
      entryPoints:
        - web
  services:
    nextcloud:
      loadBalancer:
        servers:
          - url: "http://nextcloud:80"
```

**Cloudflare Tunnel config:**
```yaml
tunnel: your-tunnel-id
credentials-file: /path/to/credentials.json

ingress:
  - hostname: nextcloud.example.com
    service: http://traefik:80
  - service: http_status:404
```

**Why this worked**: Single SSL termination point eliminates certificate chain confusion.

## 🗄️ Fix for Problem #3: Database Connection Management
**Root cause**: Nextcloud's default connection pool settings were too aggressive for my setup.

**Solution in Nextcloud config.php:**
```php
'dbdriveroptions' => [
  PDO::ATTR_TIMEOUT => 30,
  PDO::MYSQL_ATTR_INIT_COMMAND => 'SET wait_timeout = 28800'
],
'redis' => [
  'host' => 'redis',
  'port' => 6379,
  'timeout' => 3,
],
```

**PostgreSQL tuning:**
```sql
-- postgresql.conf adjustments
max_connections = 200
shared_buffers = 256MB
effective_cache_size = 1GB
```

## 🏠 Fix for Problem #4: Split-Horizon DNS with pfSense
**The elegant solution**: DNS resolver overrides in pfSense
1. Services → DNS Resolver → General Settings
2. Add Host Override:
   - Host: `nextcloud`
   - Domain: `example.com`
   - IP: `192.168.1.100` (Traefik internal IP)

**Result**: Internal clients resolve to internal IP, external clients use Cloudflare Tunnel.

---

# 📚 What I Learned

## Technical Insights
- **Docker networking is deceptively complex**: Default bridge mode works for simple setups, but custom networks are essential for production
- **SSL termination should happen once**: Multiple SSL endpoints create more problems than they solve
- **Database connection pooling matters**: Even small homelabs need proper connection management under real load
- **Split-horizon DNS is your friend**: Don't fight NAT reflection—work around it elegantly

## Process Improvements
- **Test with realistic data**: My test setup used empty Nextcloud instances; production load revealed hidden issues
- **Monitor from day one**: Setting up Prometheus/Grafana early would have caught the database issue immediately
- **Document network topology**: Drew a proper network diagram after the fact—should have done this first

## Unexpected Discoveries
- Cloudflare Tunnel's health checks can overwhelm small services if not configured properly
- Nextcloud's "trusted domains" feature is more nuanced than the documentation suggests
- pfSense's DNS resolver is incredibly powerful when you dig into the advanced options

---

# 🎯 Outcome & Next Steps

## Success Metrics
- ✅ 99.8% uptime over 6 months
- ✅ 15TB of family photos migrated from Google Photos
- ✅ Mobile apps sync reliably on WiFi and cellular
- ✅ Wife Acceptance Factor achieved (she actually prefers it now!)

## What I'd Improve Next Time
1. **Infrastructure as Code**: Convert Docker Compose to proper Kubernetes manifests
2. **Automated backups**: Implement 3-2-1 backup strategy with automated testing
3. **Monitoring stack**: Full observability with alerts for proactive issue detection
4. **Security hardening**: Implement fail2ban, regular security scans, and intrusion detection

## Future Plans
- Open source my complete setup as a "homelab-in-a-box" template
- Create Ansible playbooks for reproducible deployments
- Add Nextcloud Office integration for complete Google Workspace replacement
- Implement automated SSL certificate rotation

---

# 🧭 Related Resources / Tools

## Essential Documentation
- [Traefik Docker Provider Docs](https://doc.traefik.io/traefik/providers/docker/) - Actually read the networking section
- [Nextcloud Admin Manual](https://docs.nextcloud.com/server/latest/admin_manual/) - Performance tuning chapter is gold
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) - Focus on ingress rules

## Tools That Saved My Sanity
- [Portainer](https://www.portainer.io/) - Docker management UI
- [Netdata](https://www.netdata.cloud/) - Real-time performance monitoring
- [Docker Compose Override](https://docs.docker.com/compose/extends/) - For environment-specific configs

## My GitHub Repository
- [nextcloud-homelab-stack](https://github.com/njvanas/nextcloud-homelab-stack) - Complete configuration files and setup scripts

---

# 💬 Final Thoughts

> This project fundamentally changed how I approach homelab infrastructure. What started as a simple file server replacement became a deep dive into production-grade self-hosting. The key lesson: complexity is the enemy of reliability, but some complexity is unavoidable—the trick is managing it thoughtfully.

The three weeks of troubleshooting were frustrating in the moment, but invaluable for understanding how these systems really work under the hood. I now have a Nextcloud setup that's more reliable than many commercial services I've used, and the knowledge to replicate this success with other self-hosted applications.

Most importantly, I learned that the homelab community's emphasis on documentation and sharing knowledge isn't just nice-to-have—it's essential. Every problem I encountered had been solved by someone else, but finding those solutions required persistence and the right search terms.

Next up: tackling a self-hosted email server. How hard could it be? 😅

---

*Have questions about this setup or run into similar issues? Feel free to reach out via the contact methods in the footer, or open an issue on the GitHub repository.*