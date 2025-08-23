---
title: "Deploying a Full Servarr + qBittorrent + Plex + Jellyfin Stack on Windows with Podman"
date: 2025-03-29
tags: [homelab, podman, media-stack, servarr, windows, troubleshooting]
layout: post
---

# 📘 Overview
> This post documents how I deployed a full Servarr ecosystem (Radarr, Sonarr, Lidarr, Readarr, Prowlarr) with qBittorrent, Plex, and Jellyfin on Windows using Podman. Running this stack under Podman on Windows (via WSL2) comes with unique caveats: path mapping differences, networking quirks, and hardware transcoding limitations.

What started as a simple "let's try Podman instead of Docker" experiment turned into a deep dive into container networking, Windows-Linux path translation, and the subtle differences between container runtimes. This post chronicles the journey from "this should be straightforward" to "why can't Radarr find my downloads" to finally achieving a stable, unified media stack.

---

# 🧱 Project / Setup Context

- **Stack / Tools Used**:
  - Podman Desktop (Windows, WSL2 backend)
  - LinuxServer.io images for qBittorrent, Radarr, Sonarr, Lidarr, Readarr, Prowlarr, Plex
  - Official Jellyfin Docker image
  - Windows directories mounted into WSL via `/mnt/c/...`
  - Windows Task Scheduler for autostart

- **Initial Goals**:
  - Unified media stack in a single Pod
  - Shared download/media paths across all containers
  - Plex + Jellyfin serving the same libraries
  - Persist configs on Windows file system for easy backup
  - Replace existing Docker Desktop setup with Podman

- **Constraints**:
  - Running inside WSL2 (no direct device passthrough)
  - Windows firewall and port forwarding must be configured manually
  - No reliable hardware transcoding (Intel/NVIDIA) via WSL2
  - Limited to 16GB RAM for all containers
  - Must maintain Wife Acceptance Factor (no service interruptions)

---

# 💥 Roadblocks & Shortcomings

## 🚨 Problem #1: Path Mismatch Across Containers
- **What happened**: Each container was mounting different internal paths (`/movies`, `/tv`, `/downloads`)
- **Symptoms**: 
  - Radarr/Sonarr failed to import downloads with "path does not exist" errors
  - qBittorrent completed downloads but *arr apps couldn't find them
  - Required awkward Remote Path Mappings that kept breaking
- **Why it was frustrating**: The exact same setup worked with Docker Compose, but Podman's path handling was subtly different

## 🧩 Problem #2: Plex/Jellyfin Network Discovery Issues
- **What happened**: DLNA discovery didn't reach TVs on the LAN
- **Symptoms**:
  - Plex clients could only connect manually via IP:port
  - Jellyfin not found by smart TV apps
  - Lost auto-discovery features that worked with Docker
- **Why it was confusing**: Network settings appeared identical between Docker and Podman configurations

## 🔥 Problem #3: Hardware Transcoding Unavailable
- **What happened**: Containers couldn't access `/dev/dri` under WSL2
- **Symptoms**:
  - Only CPU transcoding available in both Plex and Jellyfin
  - Heavy CPU load during simultaneous transcodes
  - Stuttering playback on multiple concurrent streams
- **Why it was maddening**: Hardware transcoding worked fine with Docker Desktop's native Windows containers

## 🌐 Problem #4: Autostart Challenges on Windows
- **What happened**: `podman generate systemd` doesn't apply to Windows
- **Symptoms**:
  - Pod didn't restart after reboots
  - Had to manually start the entire stack each time
  - Family complained about media being "down" after Windows updates
- **Why it was painful**: Lost the seamless restart behavior from Docker Desktop

---

# 🧠 How I Resolved the Issues

## 🔧 Fix for Problem #1: Unified Path Architecture
**What didn't work:**
- Different mount points per container
- Trying to use Remote Path Mappings
- Symlinks within WSL2

**What ultimately worked:**
```yaml
# Unified host path mapping
volumes:
  - type: bind
    source: /mnt/c/Users/Dolfie/PodmanMedia
    target: /data
    
# Inside every container:
# /data/downloads/incomplete
# /data/downloads/completed  
# /data/movies
# /data/tv
# /data/music
# /data/books
```

**Key insight**: All containers must see identical paths for seamless file operations. The unified `/data` mount eliminates path translation issues entirely.

## 🛠️ Fix for Problem #2: Bridge Networking with Explicit Port Mapping
**The solution**: Switched to bridge networking with explicit `hostPort` mappings
```yaml
# Pod configuration
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: plex
      ports:
        - containerPort: 32400
          hostPort: 32400
          protocol: TCP
    - name: jellyfin
      ports:
        - containerPort: 8096
          hostPort: 8096
          protocol: TCP
```

**Acceptance**: DLNA discovery is unreliable in WSL2 environment - use Plex/Jellyfin apps with manual server configuration instead.

## 🗄️ Fix for Problem #3: CPU Transcoding Optimization
**Root cause**: WSL2 doesn't provide reliable GPU passthrough for hardware transcoding.

**Solution - Plex optimization:**
```bash
# Plex transcoding settings
- PLEX_UID=1000
- PLEX_GID=1000
- TZ=America/New_York
- PLEX_CLAIM=claim-token-here
```

**Jellyfin optimization:**
```yaml
environment:
  - JELLYFIN_PublishedServerUrl=http://192.168.1.100:8096
resources:
  limits:
    memory: 4Gi
    cpu: "2"
```

**Alternative path**: Consider running Plex for Windows natively, or migrate entire stack to Linux host for GPU support.

## 🏠 Fix for Problem #4: Windows Task Scheduler Autostart
**The elegant solution**: Windows Task Scheduler entry
1. Create new task: "Media Stack Autostart"
2. Trigger: At user logon
3. Action: Start program
   - Program: `powershell.exe`
   - Arguments: `-Command "podman play kube C:\Users\Dolfie\media-stack.yaml"`
4. Settings: Run with highest privileges

**Alternative**: Use Podman Desktop's autostart settings for individual containers.

---

# 📚 What I Learned

## Technical Insights
- **Path consistency is critical**: Unified mount points eliminate 90% of import/hardlink issues
- **WSL2 networking is different**: Bridge mode with explicit ports works better than host networking
- **Hardware transcoding requires native containers**: WSL2 adds too many abstraction layers
- **Windows autostart needs creative solutions**: Task Scheduler fills the systemd gap effectively

## Process Improvements
- **Test with realistic media files**: Empty test libraries don't reveal path mapping issues
- **Document network topology early**: Understanding WSL2's network bridge saves debugging time
- **Plan for Windows-specific quirks**: Don't assume Linux container patterns translate directly

## Unexpected Discoveries
- Podman's resource management is more granular than Docker's
- LinuxServer.io images handle PUID/PGID mapping better than official images
- Jellyfin's web interface is surprisingly responsive even with CPU-only transcoding
- Windows Defender can interfere with container file operations if not configured properly

---

# 🎯 Outcome & Next Steps

## Success Metrics
- ✅ 99.5% uptime over 3 months
- ✅ 2TB movie library fully automated via Radarr
- ✅ TV shows downloading and organizing automatically
- ✅ Both Plex and Jellyfin serving content reliably
- ✅ Family can access media from any device

## What I'd Improve Next Time
1. **Migrate to Linux host**: Enable hardware transcoding and better performance
2. **Add VPN integration**: Implement gluetun sidecar for secure downloading
3. **Implement monitoring**: Add Prometheus/Grafana for container health monitoring
4. **Automate backups**: Script regular config backups to cloud storage

## Future Plans
- Create Kubernetes manifests for more advanced orchestration
- Implement Traefik reverse proxy for unified access
- Add automated media quality management
- Document migration path from Windows to Linux deployment

---

# 🧭 Related Resources / Tools

## Essential Documentation
- [Podman Desktop Documentation](https://podman-desktop.io/docs) - Windows-specific setup guides
- [LinuxServer.io Fleet](https://fleet.linuxserver.io/) - Container image documentation
- [Servarr Wiki](https://wiki.servarr.com/) - Comprehensive setup and troubleshooting guides

## Tools That Saved My Sanity
- [Podman Desktop](https://podman-desktop.io/) - GUI management for containers
- [Windows Terminal](https://github.com/microsoft/terminal) - Better WSL2 command line experience
- [WinSCP](https://winscp.net/) - Easy file transfer between Windows and WSL2

## Configuration Files
- [media-stack-podman](https://github.com/njvanas/media-stack-podman) - Complete Pod YAML and setup scripts

---

# 💬 Final Thoughts

> This project taught me that while Podman is an excellent Docker alternative, Windows deployment comes with unique challenges that require creative solutions. The key lesson: embrace the constraints rather than fighting them, and design your architecture around the platform's strengths.

The three weeks of path mapping troubleshooting were frustrating, but resulted in a deeper understanding of how container filesystems work across different runtimes. I now have a media stack that's more resource-efficient than my previous Docker setup, with better isolation and security.

Most importantly, I learned that the homelab journey is about continuous iteration. This Windows Podman setup serves as an excellent stepping stone toward a future Linux deployment with full hardware acceleration, while providing immediate value to my family's media consumption needs.

Next challenge: implementing a proper VPN solution for secure downloading. The adventure continues! 🚀

---

*Running into similar issues with Podman on Windows or have questions about this setup? Feel free to reach out via the contact methods in the footer, or check out the complete configuration files in the GitHub repository.*