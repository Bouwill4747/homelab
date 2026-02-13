**Built by a cybersecurity student who wanted to learn infrastructure security hands-on - not just read about it.**

  ---

  ## Architecture Overview

  | Node | Role | CPU | RAM | GPU | Storage |
  |------|------|-----|-----|-----|---------|
  | **pve1** | Core Infrastructure | i5-9600K | 32GB | RTX 2070 | 2x6TB HDD + 2TB NVMe |
  | **pve2** | AI/GPU Workloads | i5-9600K | 32GB | RTX 3090 | 500GB + 1TB NVMe |
  | **pve3** | Media/Security | i7-9700KF | 32GB | RTX 3080 | 500GB NVMe + 2TB SATA |

  **Network:** ISP Router (bridged) → OPNsense VM (firewall/gateway) → Netgear GS110TPv3 switch → all nodes

  ## Services (13 Total)

  ### Infrastructure
  | Service | Type | Description |
  |---------|------|-------------|
  | **OPNsense** | VM | Firewall, router, gateway - with CrowdSec IDS/IPS and TOTP 2FA |
  | **AdGuard Home** | LXC | Network-wide DNS filtering - blocks ads, trackers, and malware domains |
  | **TrueNAS SCALE** | VM | NAS with 6TB ZFS mirror, NFS shares for cluster storage |
  | **Caddy** | LXC | Reverse proxy with .lan hostnames for all services |
  | **Proxmox Backup Server** | VM | Automated daily backups with deduplication and retention policies |

  ### Security & Monitoring
  | Service | Type | Description |
  |---------|------|-------------|
  | **Wazuh SIEM** | VM | Centralized log collection, threat detection, vulnerability scanning - agents on all nodes |
  | **CrowdSec** | OPNsense Plugin | Collaborative IDS/IPS with community threat intelligence blocklists |
  | **Uptime Kuma** | LXC | Service availability monitoring with alerting |
  | **Beszel** | LXC | Server resource monitoring (CPU, RAM, disk, network) across all nodes |

  ### Media & Automation
  | Service | Type | Description |
  |---------|------|-------------|
  | **Jellyfin** | VM | Media server with RTX 3080 GPU passthrough for NVENC hardware transcoding |
  | **Radarr / Sonarr / Prowlarr** | VM | Automated media management with indexer integration |
  | **qBittorrent + Gluetun** | VM | Torrent client routed through ProtonVPN - built-in kill switch via Docker network isolation |

  ### Management & Access
  | Service | Type | Description |
  |---------|------|-------------|
  | **Homepage** | LXC | Unified dashboard for all services |
  | **Portainer** | LXC | Centralized Docker container management across multiple hosts |
  | **Tailscale** | Host | Secure remote access via WireGuard mesh VPN — zero open ports |

  ---

  ## Security Architecture

  ### Defense-in-Depth Layers

  Internet
      │
      ▼
  ┌─────────────────────────┐
  │  OPNsense Firewall      │  ← All inbound traffic blocked by default
  │  + CrowdSec IDS/IPS     │  ← Community blocklists + behavioral detection
  │  + TOTP 2FA             │  ← Two-factor auth on admin access
  └─────────────────────────┘
      │
      ▼
  ┌─────────────────────────┐
  │  AdGuard Home DNS       │  ← Blocks malware/phishing domains network-wide
  └─────────────────────────┘
      │
      ▼
  ┌─────────────────────────┐
  │  Wazuh SIEM             │  ← Log correlation, file integrity monitoring,
  │  (agents on all nodes)  │     vulnerability detection, MITRE ATT&CK mapping
  └─────────────────────────┘
      │
      ▼
  ┌─────────────────────────┐
  │  VPN Kill Switch        │  ← All torrent traffic through ProtonVPN
  │  (Gluetun + Docker)     │  ← If VPN drops, network dies — zero IP leaks
  └─────────────────────────┘
      │
      ▼
  ┌─────────────────────────┐
  │  Tailscale              │  ← Remote access via encrypted WireGuard tunnels
  │  (zero open ports)      │  ← No port forwarding needed on firewall
  └─────────────────────────┘  Internet
      │
      ▼
  ┌─────────────────────────┐
  │  OPNsense Firewall      │  ← All inbound traffic blocked by default
  │  + CrowdSec IDS/IPS     │  ← Community blocklists + behavioral detection
  │  + TOTP 2FA             │  ← Two-factor auth on admin access
  └─────────────────────────┘
      │
      ▼
  ┌─────────────────────────┐
  │  AdGuard Home DNS       │  ← Blocks malware/phishing domains network-wide
  └─────────────────────────┘
      │
      ▼
  ┌─────────────────────────┐
  │  Wazuh SIEM             │  ← Log correlation, file integrity monitoring,
  │  (agents on all nodes)  │     vulnerability detection, MITRE ATT&CK mapping
  └─────────────────────────┘
      │
      ▼
  ┌─────────────────────────┐
  │  VPN Kill Switch        │  ← All torrent traffic through ProtonVPN
  │  (Gluetun + Docker)     │  ← If VPN drops, network dies — zero IP leaks
  └─────────────────────────┘
      │
      ▼
  ┌─────────────────────────┐
  │  Tailscale              │  ← Remote access via encrypted WireGuard tunnels
  │  (zero open ports)      │  ← No port forwarding needed on firewall
  └─────────────────────────┘
  └─────────────────────────┘

  ### What's Exposed vs. Protected

  | Component | Exposed to Internet? | Protection |
  |-----------|---------------------|------------|
  | OPNsense WAN | Public IP visible | Firewall drops all unsolicited inbound |
  | All internal services | No | Only reachable on LAN or via Tailscale |
  | Torrent traffic | ProtonVPN IP only | Real IP never exposed to peers |
  | Remote access | No open ports | Tailscale NAT traversal + WireGuard encryption |
  | Admin interfaces | No | LAN-only + 2FA on OPNsense |

  ---

  ## Notable Challenges & Solutions

  ### PBS Backup Crashed GPU Passthrough Host
  **Problem:** Proxmox Backup Server's snapshot-based backup of a VM with RTX 3090 GPU passthrough caused a kernel panic on the host - hard reboot with no shutdown logs.

  **Root Cause:** VM snapshots briefly freeze I/O. GPU drivers can't handle this freeze, triggering a host-level crash.

  **Solution:** Excluded all GPU passthrough VMs from snapshot-based backups. These VMs require alternative backup strategies (application-level backups, config exports).

  **Lesson:** Never snapshot VMs with PCIe passthrough devices. This also applied to TrueNAS with disk passthrough - snapshots caused ZFS pool degradation.

  ---

  ### VPN Container Failed in LXC - TUN Device Blocked
  **Problem:** Gluetun VPN container couldn't access `/dev/net/tun` inside a Proxmox LXC container, even with privileged mode and AppArmor disabled.

  **Root Cause:** Multiple security layers (cgroups, AppArmor, LXC namespacing) block TUN device access in ways that are extremely difficult to work around.

  **Solution:** Converted the LXC to a full VM. VMs have complete kernel access, so TUN devices work natively.

  **Lesson:** If you need VPN containers in Proxmox, use a VM, not an LXC.

  ---

  ### Wazuh Agent Version Mismatch
  **Problem:** Wazuh agents (v4.14.3) couldn't register with the manager (v4.11.2) - "Agent version must be lower or equal to manager version."

  **Root Cause:** The default apt repository installed the latest agent, but the all-in-one installer deployed an older manager.

  **Solution:** Pinned agent version to match the manager: `apt install wazuh-agent=4.11.2-1`

  **Lesson:** Always verify version compatibility between SIEM components before deployment.

  ---

  ## Skills Demonstrated

  - **Network Security:** Firewall configuration, IDS/IPS deployment, VPN implementation, DNS filtering
  - **SIEM Operations:** Wazuh deployment, agent management, log correlation, threat detection
  - **Virtualization:** Proxmox VE clustering, VM/LXC management, GPU passthrough, storage passthrough
  - **Storage:** ZFS mirror pools, NFS shared storage, backup strategies with retention policies
  - **Containerization:** Docker, Docker Compose, multi-service orchestration, Portainer management
  - **Linux Administration:** Debian, FreeBSD (OPNsense), systemd, networking, troubleshooting
  - **Infrastructure Monitoring:** Service uptime, resource utilization, alerting
  - **Documentation:** Operational logs, change management, incident tracking

  ---

  ## Implementation Phases

  | Phase | Focus | Status |
  |-------|-------|--------|
  | 1 | Foundation - repos, static IPs, cluster, OPNsense, AdGuard
  | 2 | Storage - TrueNAS SCALE, ZFS import, NFS shares
  | 3 | Core Services - Jellyfin (GPU), Caddy, Homepage
  | 4 | Quality of Life - Uptime Kuma, Beszel, Tailscale
  | 5 | Enhancement - *arr stack, PBS, CrowdSec
  | 6 | Security & Management - Wazuh SIEM, Portainer 

  ---

  ## Future Improvements

  - **Network Segmentation (VLANs)** - Isolate IoT, servers, and personal devices to limit lateral movement
  - **Ansible Automation** - Infrastructure as code for reproducible deployments
  - **Kali Linux Security Lab** - Penetration testing practice environment
  - **Authentik SSO** - Single sign-on across all services
  - **Domain + HTTPS** - Real TLS certificates via Caddy with DNS challenge
