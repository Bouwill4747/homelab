# Security Hardening

> Every security measure implemented on this homelab, organized by defense-in-depth layer. This isn't a checklist — each section explains what the control does and why it matters.

---

## Layer 1: Perimeter Defense

### OPNsense Firewall (VM 100)
- **Role:** All traffic between the internet and the LAN passes through OPNsense
- **Default policy:** Deny all inbound, allow established/related outbound
- **DHCP server:** OPNsense manages all IP assignments (10.0.0.50-200 range)
- **MAC spoofing:** WAN interface clones a known MAC to satisfy ISP binding
- **2FA enabled:** TOTP (Time-based One Time Password) on admin login — password alone isn't enough

### CrowdSec IDS/IPS
- **Installed on:** OPNsense via `os-crowdsec` plugin
- **What it does:** Monitors OPNsense logs in real-time for attack patterns (brute force, port scans, exploit attempts)
- **Automatic blocking:** Detected attackers get firewall block rules created automatically
- **Community intelligence:** Registered with CrowdSec Central API — receives crowd-sourced IP blocklists from thousands of other CrowdSec users worldwide
- **Why CrowdSec over Suricata:** Lighter on resources, crowd-sourced model provides broader threat coverage, better suited for homelab scale

### Zero Open Ports
- No port forwarding configured on OPNsense
- All remote access goes through Tailscale (WireGuard mesh VPN)
- External port scans see nothing — the attack surface is effectively zero

---

## Layer 2: DNS Filtering

### AdGuard Home (CT 101)
- **Role:** All DNS queries on the network pass through AdGuard before resolving
- **Blocks:** Ads, trackers, malware domains, phishing sites, telemetry
- **Integration:** OPNsense DHCP pushes AdGuard (10.0.0.3) as the DNS server to all clients
- **DNS rewrites:** All `*.lan` hostnames resolve locally (never leave the network)
- **Why DNS filtering matters:** Blocks threats at the earliest possible stage — before the connection is even established. A device can't download malware from a domain it can't resolve.

---

## Layer 3: Network Monitoring & SIEM

### Wazuh SIEM (VM 301)
- **Components:** Manager (event processor) + Indexer (log storage) + Dashboard (web UI)
- **Agents deployed on:** pve1, pve2, pve3 (all Proxmox hosts)
- **Capabilities:**
  - File integrity monitoring (detects unauthorized file changes)
  - Rootkit detection
  - Log analysis and correlation
  - Vulnerability detection
  - Security event alerting
- **Why Wazuh:** Industry-standard open-source SIEM, same platform used in enterprise SOCs. Practical experience that translates directly to cybersecurity careers.

### Uptime Kuma (CT 106)
- Monitors all services (HTTP checks) and all nodes (PING checks)
- Detects outages within 60 seconds
- Provides uptime history and response time tracking

### Beszel (CT 107)
- Lightweight system metrics collection (CPU, RAM, disk, network)
- Agents on all 3 Proxmox hosts
- Historical data for capacity planning and anomaly detection

---

## Layer 4: Application Security

### VPN Kill Switch (*arr Stack)
- All torrent traffic routes through Gluetun VPN container (ProtonVPN)
- Docker `network_mode: service:gluetun` ensures containers CANNOT bypass the VPN
- If the VPN connection drops, all traffic stops immediately — real IP is never exposed
- ProtonVPN port forwarding enabled for seeding (outbound only, through VPN tunnel)

### Reverse Proxy Isolation (Caddy)
- Backend services are not directly exposed — all access goes through Caddy
- Services bind to internal IPs only
- Host header validation prevents request smuggling

### Tailscale Remote Access
- WireGuard-based mesh VPN — encrypted end-to-end
- NAT traversal without opening any firewall ports
- Identity-based authentication (tied to SSO provider)
- Deployed on: Jellyfin VM, pve2 host (LLM access), pve1 host (jump box to entire network)

---

## Layer 5: Backup & Recovery

### Proxmox Backup Server (VM 109)
- **Schedule:** Daily at 02:00
- **Mode:** Snapshot (zero-downtime)
- **Storage:** TrueNAS NFS share (physically separate disks)
- **Retention:** 7 days
- **Exclusions:** VM 103 (TrueNAS — disk passthrough), VM 200 (LLM — GPU passthrough)
  - *Learned the hard way: snapshot-based backups cause kernel panics on passthrough VMs*

### TrueNAS ZFS Mirror
- 2x6TB WD Red Plus in ZFS mirror configuration
- Automatic checksumming detects silent data corruption (bit rot)
- If one drive fails, the mirror has a complete copy
- SMART monitoring for early drive failure detection

---

## Layer 6: Host Security

### Proxmox Cluster
- 3-node cluster with quorum (requires 2/3 nodes for HA decisions)
- Prevents split-brain scenarios
- Encrypted cluster communication via Kronosnet (knet)

### Repository Configuration
- All nodes switched from enterprise (paid) to community (free) repositories
- Ensures security patches are still received without subscription

### Container Isolation
- LXC containers run unprivileged by default (reduced kernel attack surface)
- `nesting=1` only enabled where Docker is required
- VMs used for workloads requiring elevated privileges (VPN, GPU passthrough)

---

## Security Architecture Summary

| Layer | Control | Threat Mitigated |
|-------|---------|-----------------|
| Perimeter | OPNsense + CrowdSec + 2FA | Unauthorized access, brute force, known malicious IPs |
| DNS | AdGuard Home | Malware domains, phishing, tracking, ads |
| Monitoring | Wazuh SIEM | Undetected intrusions, file tampering, vulnerabilities |
| Application | VPN kill switch, Tailscale | IP exposure, man-in-the-middle, unauthorized remote access |
| Backup | PBS + ZFS mirror | Ransomware, data loss, hardware failure |
| Host | Unprivileged LXC, cluster quorum | Privilege escalation, split-brain |

---

## What I Would Add Next

- **Wazuh agents on VMs/containers** (not just Proxmox hosts) for deeper visibility
- **Fail2ban** on services with authentication (extra layer beyond CrowdSec)
- **Automated vulnerability scanning** with OpenVAS or similar
- **Network segmentation** (VLANs) if IoT devices are ever added
- **Encrypted backups** for sensitive data stored on PBS
- **Log forwarding** from all services to Wazuh for centralized correlation
