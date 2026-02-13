# Lessons Learned & Troubleshooting Log

> Real problems encountered during the build, how I diagnosed them, and what I learned. This is the messy, honest side of infrastructure work that tutorials never show you.

---

## 1. Cluster Join Blocked by Existing VM Configs

**Phase:** Foundation (Cluster Formation)
**Severity:** Blocking

**What happened:** Running `pvecm add` on pve2 to join the cluster failed with:
```
this host already contains virtual guests
```

**Root cause:** Proxmox replaces `/etc/pve` with the cluster's shared filesystem during join. If the node already has VM or LXC configs in `/etc/pve/qemu-server/` or `/etc/pve/lxc/`, it refuses to proceed to avoid silently destroying those configs.

**Fix:**
1. Backed up the existing VM 200 config
2. Removed the LXC config from `/etc/pve/lxc/`
3. Joined the cluster successfully
4. Re-added the local storage pool: `pvesm add lvmthin nvme-fast --thinpool thinpool --vgname nvme-fast --nodes pve2`
5. Restored VM 200 config

**Lesson:** Always audit `/etc/pve/qemu-server/` AND `/etc/pve/lxc/` before joining a cluster. Local storage pools also need to be re-registered after join.

---

## 2. Cluster Join SSL Authentication Failure

**Phase:** Foundation (Cluster Formation)
**Severity:** Blocking

**What happened:** `pvecm add 10.0.0.115` on pve3 failed with SSL authentication errors.

**Fix:** Used `pvecm add 10.0.0.115 --use_ssh` to bypass TLS and use SSH-based authentication instead.

**Lesson:** The `--use_ssh` flag is a standard workaround when certificate trust isn't established between nodes before join. Doesn't affect cluster security after join is complete.

---

## 3. Cable Negotiation at 100 Mbps Instead of 1 Gbps

**Phase:** Foundation (Network)
**Severity:** Performance

**What happened:** Speed tests showed 90-95 Mbps. Link speed showed 100/100 Mbps instead of the expected 1000/1000 Mbps. Both pve1 NIC ports confirmed at 1 Gbps, so the bottleneck was between the PC and the switch.

**Root cause:** Gigabit Ethernet requires all 4 pairs (8 wires). If any pair has weak contact from a partial seat or pin oxidation, the link falls back to Fast Ethernet (100 Mbps) which only needs 2 pairs.

**Fix:** Unplugged and re-seated the same cable. Link renegotiated at 1 Gbps immediately.

**Lesson:** If speeds are capped at ~95 Mbps, check the physical layer first. A cable reseat is the easiest fix in networking.

---

## 4. ISP Bridge Mode - No IP Assigned (MAC Binding)

**Phase:** Foundation (OPNsense Deployment)
**Severity:** Blocking

**What happened:** After enabling bridge mode on the ISP router, OPNsense WAN received no IPv4 address. A laptop plugged directly in also got nothing — confirmed it wasn't OPNsense's fault.

**Root cause:** ISP MAC address binding. In bridge mode, the ISP only provisions an IP to a MAC address it recognizes. It saw an unknown device and refused to issue an IP.

**Fix:** Spoofed the laptop's MAC address on OPNsense's WAN interface (Interfaces > WAN > MAC address). Also required calling the ISP to confirm bridge mode was enabled on their end.

**Lesson:** If an ISP won't hand out an IP in bridge mode, MAC binding is the most common cause. Clone the MAC of a device the ISP has previously seen.

---

## 5. Docker in Unprivileged LXC - Permission Denied

**Phase:** Core Services
**Severity:** Blocking

**What happened:** `docker run` inside an unprivileged LXC failed with:
```
OCI runtime create failed: error mounting proc: permission denied
```

**Root cause:** Docker needs to create nested namespaces, which unprivileged LXC blocks by default for security.

**Fix:** `pct set <id> --features nesting=1` then restart the container.

**Lesson:** Always enable `nesting=1` when planning to run Docker inside an unprivileged LXC on Proxmox. This is a one-time setting per container.

---

## 6. VPN TUN Device Not Available in LXC

**Phase:** Enhancement (*arr Stack)
**Severity:** Blocking — required architecture change

**What happened:** Gluetun VPN container failed with:
```
TUN device is not available: open /dev/net/tun: operation not permitted
```
This happened in both unprivileged AND privileged LXC containers.

**What I tried (all failed):**
- cgroup2 device allow rules
- Bind mounting `/dev/net/tun`
- Privileged container mode
- AppArmor `unconfined` profile

**Root cause:** Multiple layers of security (cgroups, AppArmor, LXC namespacing) block Docker containers from accessing `/dev/net/tun` even when the device exists.

**Fix:** Converted the LXC to a full VM. VMs have full kernel access, so TUN devices work natively.

**Lesson:** If you need VPN containers (Gluetun, WireGuard) in Proxmox, use a VM, not an LXC. The TUN device restrictions in LXC are too complex to reliably work around. This cost several hours of debugging before arriving at this conclusion.

---

## 7. Gluetun VPN DNS Resolution Failure

**Phase:** Enhancement (*arr Stack)
**Severity:** Service-affecting

**What happened:** Gluetun connected to ProtonVPN successfully, but DNS lookups inside the tunnel failed. The VPN was up, but no container could resolve hostnames.

**Fix:** Set Gluetun environment variables:
```
DNS_ADDRESS=10.2.0.1    # ProtonVPN's internal DNS server
DOT=off                  # Disable DNS-over-TLS (conflicts with VPN DNS routing)
```

Also added health check tuning to prevent restart loops:
```
HEALTH_VPN_DURATION_INITIAL=30s
HEALTH_SUCCESS_WAIT_DURATION=20s
```

**Lesson:** VPN containers often need explicit DNS configuration. The default DNS resolver may not work inside the tunnel. Check your VPN provider's documentation for their internal DNS server address.

---

## 8. PBS Backup Degraded TrueNAS ZFS Pool

**Phase:** Enhancement (Backups)
**Severity:** Critical — data at risk

**What happened:** After the PBS backup job ran, TrueNAS reported pool "tank" as DEGRADED. One disk FAULTED, one DEGRADED. NFS shares went down, breaking the *arr stack.

**Root cause:** PBS uses Proxmox VM snapshots to back up running VMs. Snapshots briefly freeze ALL I/O — including to passthrough disks. ZFS inside TrueNAS interpreted the I/O freeze as physical disk failure and faulted the drives.

**Fix:**
1. `zpool clear tank` — restored the pool to ONLINE (no data lost thanks to mirror redundancy)
2. Excluded VM 103 (TrueNAS) from PBS backup job permanently

**Lesson:** NEVER include VMs with physical disk passthrough in snapshot-based backups. ZFS + disk passthrough + VM snapshots don't mix. Back up TrueNAS separately using its built-in replication/config export tools.

---

## 9. PBS Backup Crashed pve2 (GPU Passthrough VM)

**Phase:** Enhancement (Backups)
**Severity:** Critical — host crash

**What happened:** pve2 hard-rebooted at 03:08 with no clean shutdown in logs. Journals show PBS unreachable starting at 00:32, then logs stop abruptly — no systemd shutdown sequence. This was a kernel panic.

**Root cause:** PBS backup job at 02:00 attempted to snapshot VM 200 (LLM with RTX 3090 passthrough). GPU drivers can't handle the I/O freeze from snapshots, causing a kernel panic on the host.

**Fix:** Excluded VM 200 from PBS backup job.

**Lesson:** NEVER snapshot VMs with GPU passthrough. Both GPU passthrough and disk passthrough VMs must be excluded from PBS. All passthrough VMs need alternative backup strategies.

---

## 10. Wazuh Agent Version Mismatch

**Phase:** Security (Wazuh SIEM)
**Severity:** Service-affecting

**What happened:** Installed Wazuh agents on all 3 Proxmox nodes, but none appeared in the Wazuh dashboard. Agent logs showed connection failures.

**Root cause:** `apt install wazuh-agent` pulled version 4.14.3 (latest in repo), but the Wazuh manager was version 4.11.2. Wazuh requires agents and manager to be the same version.

**Fix:**
```bash
apt install wazuh-agent=4.11.2-1
sed -i 's/MANAGER_IP/10.0.0.14/' /var/ossec/etc/ossec.conf
systemctl restart wazuh-agent
```

**Lesson:** Always pin Wazuh agent version to match your manager version during installation. The default `apt install` grabs the latest, which may not be compatible.

---

## 11. Homepage Host Validation Failed

**Phase:** Core Services
**Severity:** Service-affecting

**What happened:** Homepage returned "Host validation failed" when accessed through the Caddy reverse proxy, even with `allowedHosts` configured in `settings.yaml`.

**Root cause:** Newer versions of Homepage require the `HOMEPAGE_ALLOWED_HOSTS` environment variable instead of the YAML config.

**Fix:** Recreated the container with:
```bash
docker run -d --name homepage -e HOMEPAGE_ALLOWED_HOSTS=homepage.lan,10.0.0.7,10.0.0.4 ...
```

**Lesson:** When a Docker app has host validation issues, check the container logs first — they often hint at whether the fix is an environment variable or config file change.

---

## 12. OPNsense Throughput Degradation (Previous Setup)

**Phase:** Pre-project (lesson applied to new build)
**Severity:** Performance

**What happened:** In a previous setup, 900 Mbps direct to ISP dropped to 300 Mbps when routing through a virtualized OPNsense.

**Root cause:** No PCIe passthrough of the NIC — traffic went through Proxmox's Linux bridge, adding double network stack overhead. Possibly compounded by emulated NICs (e1000 vs VirtIO) and IDS/IPS running.

**How I applied this lesson:**
- Originally planned PCIe passthrough for the new build
- Discovered IOMMU grouping prevented it (NIC shared group with GPU)
- Used VirtIO NICs instead with proper tuning (host CPU type, 4 cores)
- Result: ~900 Mbps throughput achieved — VirtIO on 1GbE was sufficient

**Lesson:** PCIe passthrough is ideal but not always possible due to IOMMU groups. VirtIO with proper configuration can achieve near-line-speed on 1GbE networks.

---

## Key Takeaways

1. **Physical layer first** — Before debugging software, check cables, link speeds, and connectivity
2. **LXC vs VM** — Use VMs for anything needing TUN devices, GPU passthrough, or disk passthrough. LXC for everything else
3. **Snapshot-based backups are dangerous** with passthrough hardware — always exclude these VMs
4. **Version pinning matters** — Enterprise software like Wazuh requires exact version matching between components
5. **ISPs do weird things** — MAC binding, requiring phone calls to enable features, and non-standard bridge mode behavior are all common
6. **Read the logs** — Nearly every issue was diagnosable from container logs, systemd journals, or ZFS status output
