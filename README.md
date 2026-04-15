# Secure Home Lab Network Architecture

A production-grade home lab environment built on Proxmox, designed to emulate enterprise network segmentation, threat detection, and secure remote access. This project documents the full architecture, design rationale, and implementation details.

## Hardware

| Device | Model | Role | Notes |
|--------|-------|------|-------|
| Hypervisor | Dell OptiPlex 7080 Micro (i7-10700, 64GB DDR4, 1TB NVMe + 2TB SATA SSD) | Proxmox VE 8.1 host | Runs all VMs. Dual NIC — onboard Intel I219-LM for LAN trunk, added Intel I350-T2 for dedicated WAN passthrough to pfSense. |
| Core Switch | Ubiquiti UniFi Switch 16 PoE (USW-16-POE) | Managed L2 switch, 802.1Q VLAN trunking | 16 GbE ports. Trunk port 1 uplinks to Proxmox. Access ports assigned per VLAN. |
| Access Point | Ubiquiti UniFi U6 Lite | Wireless for Trusted + IoT VLANs | Broadcasts two SSIDs — `RAINE-TRUSTED` (VLAN 10, WPA3) and `RAINE-IOT` (VLAN 30, WPA2). No wireless access to Lab or Management VLANs. |
| UPS | CyberPower CP1500AVRLCD | Battery backup | Provides ~18 min runtime. USB connected to Proxmox for automated graceful shutdown via `apcupsd`. |

## Architecture Overview

```
                            ┌──────────────┐
                            │   INTERNET   │
                            └──────┬───────┘
                                   │
                            ┌──────┴───────┐
                            │  Arris SB8200 │
                            │ (Bridge Mode) │
                            │  WAN IP: DHCP │
                            └──────┬───────┘
                                   │ (Intel I350 — PCI passthrough)
                      ┌────────────┴────────────┐
                      │      pfSense 2.7.2       │
                      │   (Proxmox VM — 4C/8GB)  │
                      │                          │
                      │  WAN: em0 — DHCP         │
                      │  LAN: vtnet0 — 10.10.0.1 │
                      │  Suricata on LAN + VLANs │
                      │  ntopng on LAN trunk     │
                      │  Unbound DNS (DoT → 1.1.1.1)
                      └────────────┬────────────┘
                                   │ trunk (tagged VLANs 10,20,30,99)
                      ┌────────────┴────────────┐
                      │   USW-16-POE (Port 1)   │
                      │   Ubiquiti UniFi Switch │
                      └─┬───────┬───────┬──────┬┘
                        │       │       │       \
                        |       |       |         \
              ┌─────────┴┐ ┌───-┴────┐ ┌┴──────-──┐ ┌───────────┐
              │ VLAN 10   │ │VLAN 20 │ │ VLAN 30  │ │  VLAN 99  │
              │ TRUSTED   │ │  LAB   │ │   IOT    │ │ MANAGEMENT│
              │10.10.10.0 │ │10.10.20│ │10.10.30.0│ │ 10.10.99.0│
              │   /24     │ │  /24   │ │   /24    │ │    /24    │
              │           │ │        │ │          │ │           │
              │ Gateway:  │ │Gateway:│ │ Gateway: │ │ Gateway:  │
              │ .1        │ │  .1    │ │   .1     │ │   .1      │
              │ DHCP:     │ │ DHCP:  │ │ DHCP:    │ │ Static    │
              │ .100-.200 │ │.100-150│ │ .100-.200│ │ only      │
              └───────────┘ └────────┘ └──────────┘ └───────────┘
```

## Network Segments

### VLAN 10 — Trusted (10.10.10.0/24)

Primary workstations and personal devices. Full outbound internet access with DNS filtering via Unbound (blocks ad/tracking domains using a curated blocklist). This is the only VLAN permitted to initiate SSH connections to VLAN 99 (Management).

| IP Address | Device | Notes |
|------------|--------|-------|
| 10.10.10.1 | pfSense (gateway) | VLAN 10 interface |
| 10.10.10.10 | Primary workstation (Desktop — Ubuntu 24.04) | Static DHCP lease, SSH enabled |
| 10.10.10.11 | Laptop (Framework 13 — Fedora 40) | Static DHCP lease |
| 10.10.10.12 | Desktop PC (Windows — daily driver) | Static DHCP lease |
| 10.10.10.100–.200 | DHCP pool | Guest devices, phones |

**Switch port assignments:** Ports 2–5 on the USW-16-POE are tagged VLAN 10. The U6 Lite AP on port 14 (PoE) carries both VLAN 10 (`RAINE-TRUSTED`) and VLAN 30 (`RAINE-IOT`) SSIDs.

### VLAN 20 — Lab (10.10.20.0/24)

Security testing and development environment. Fully isolated from Trusted and IoT segments. Runs vulnerable target VMs for offensive security practice. Outbound internet is filtered and logged — all DNS goes through a local Unbound instance within the VLAN to prevent DNS-based data exfiltration during red team exercises.

| IP Address | Device / VM | Notes |
|------------|-------------|-------|
| 10.10.20.1 | pfSense (gateway) | VLAN 20 interface |
| 10.10.20.10 | Kali Linux (Proxmox VM — 4C/8GB) | Primary attack box |
| 10.10.20.11 | Metasploitable 2 (Proxmox VM — 1C/1GB) | Intentionally vulnerable target |
| 10.10.20.12 | DVWA — Damn Vulnerable Web App (Proxmox LXC) | Web app exploitation practice |
| 10.10.20.13 | VulnHub — Kioptrix L1 (Proxmox VM — 1C/512MB) | Boot-to-root challenge |
| 10.10.20.14 | Windows Server 2019 (Proxmox VM — 2C/4GB) | AD lab — domain controller |
| 10.10.20.15 | Windows 10 (Proxmox VM — 2C/4GB) | AD lab — domain-joined workstation |
| 10.10.20.50 | Unbound DNS resolver (Proxmox LXC — 1C/256MB) | Local-only DNS for lab isolation |
| 10.10.20.100–.150 | DHCP pool | Temporary VMs spun up for CTFs |

**Switch port assignments:** Ports 6–8 tagged VLAN 20. No wireless access — lab is wired only.

**Active Directory sub-lab:** The Windows Server 2019 DC (`LAB.LOCAL`) and the domain-joined Windows 10 box are used for practicing Kerberoasting, Pass-the-Hash, BloodHound enumeration, and Group Policy abuse. These are snapshot-and-revert environments — they get rolled back to a clean state after each exercise.

### VLAN 30 — IoT (10.10.30.0/24)

Isolated network for IoT devices. No inter-VLAN routing permitted — IoT devices cannot initiate connections to any other segment. Outbound access is restricted to specific ports and destinations only. All traffic is monitored by Suricata with IoT-specific rulesets.

| IP Address | Device | Notes |
|------------|--------|-------|
| 10.10.30.1 | pfSense (gateway) | VLAN 30 interface |
| 10.10.30.10 | Philips Hue Bridge | Static lease. Outbound: TCP 443 to `*.meethue.com` only |
| 10.10.30.11 | Roku Ultra | Static lease. Outbound: TCP 443/80, UDP 53 |
| 10.10.30.12 | Ecobee Smart Thermostat | Static lease. Outbound: TCP 443 to `*.ecobee.com` only |
| 10.10.30.13–.15 | TP-Link Kasa Smart Plugs (×3) | Static leases. Outbound: TCP 443, UDP 123 (NTP) |
| 10.10.30.100–.200 | DHCP pool | New IoT devices land here with restricted access until explicitly allowed |

**Switch port assignments:** Ports 9–11 tagged VLAN 30. Wireless access via `RAINE-IOT` SSID on the U6 Lite (VLAN 30 tagged).

**Why isolate IoT?** Consumer IoT devices are notorious for phoning home to unexpected endpoints, having unpatched firmware, and running insecure protocols. Isolating them means a compromised smart plug can't be used to pivot into my workstation or lab. The per-device outbound rules were built by first putting each device in monitor mode with ntopng, watching what it actually talks to over a week, and then writing allow rules for only those destinations.

### VLAN 99 — Management (10.10.99.0/24)

Administrative access only. No DHCP — every device has a manually assigned static IP. Accessible only from VLAN 10 via explicit firewall rules on specific ports.

| IP Address | Service | Access Port |
|------------|---------|-------------|
| 10.10.99.2 | Proxmox VE Web UI | TCP 8006 |
| 10.10.99.3 | pfSense Admin Panel | TCP 443 |
| 10.10.99.4 | UniFi Network Controller (Proxmox LXC) | TCP 8443 |
| 10.10.99.5 | ntopng Dashboard (Proxmox LXC) | TCP 3000 |
| 10.10.99.6 | Syslog Aggregator — rsyslog (Proxmox LXC — 1C/512MB) | UDP 514, TCP 514 |

**Switch port assignments:** Port 16 tagged VLAN 99 (dedicated management uplink). No wireless access — management VLAN is wired and accessible only via VLAN 10.

## Proxmox VM / LXC Inventory

| VMID | Name | Type | vCPU | RAM | Disk | VLAN | Status |
|------|------|------|------|-----|------|------|--------|
| 100 | pfSense | VM | 4 | 8 GB | 32 GB | WAN + trunk | Running (autostart) |
| 101 | Kali Linux | VM | 4 | 8 GB | 80 GB | 20 | Running |
| 102 | Metasploitable 2 | VM | 1 | 1 GB | 8 GB | 20 | Stopped (on-demand) |
| 103 | DVWA | LXC | 1 | 512 MB | 4 GB | 20 | Stopped (on-demand) |
| 104 | Kioptrix L1 | VM | 1 | 512 MB | 8 GB | 20 | Stopped (on-demand) |
| 105 | WinServer 2019 DC | VM | 2 | 4 GB | 60 GB | 20 | Stopped (on-demand) |
| 106 | Win10 Workstation | VM | 2 | 4 GB | 50 GB | 20 | Stopped (on-demand) |
| 107 | Lab DNS (Unbound) | LXC | 1 | 256 MB | 2 GB | 20 | Running (autostart) |
| 200 | UniFi Controller | LXC | 1 | 1 GB | 8 GB | 99 | Running (autostart) |
| 201 | ntopng | LXC | 2 | 2 GB | 20 GB | 99 | Running (autostart) |
| 202 | Syslog (rsyslog) | LXC | 1 | 512 MB | 30 GB | 99 | Running (autostart) |

**Storage layout:** Proxmox host has two drives. The 1TB NVMe (`local-zfs`) stores VM disks for anything performance-sensitive (pfSense, Kali). The 2TB SATA SSD (`data-zfs`) stores LXC containers, backups, and ISO images. ZFS snapshots run daily at 2 AM and retain 7 days.

## Firewall Rules

Rules follow a default-deny model. Every interface starts with an implicit "block all" at the bottom. Only explicitly permitted traffic passes.

### VLAN 10 (Trusted) — Outbound & Inter-VLAN

| # | Action | Proto | Source | Dest | Port | Description |
|---|--------|-------|--------|------|------|-------------|
| 1 | Pass | TCP | 10.10.10.0/24 | 10.10.99.2 | 8006 | Allow Proxmox UI access from Trusted |
| 2 | Pass | TCP | 10.10.10.0/24 | 10.10.99.3 | 443 | Allow pfSense admin from Trusted |
| 3 | Pass | TCP | 10.10.10.0/24 | 10.10.99.4 | 8443 | Allow UniFi controller from Trusted |
| 4 | Pass | TCP | 10.10.10.0/24 | 10.10.99.5 | 3000 | Allow ntopng dashboard from Trusted |
| 5 | Pass | TCP/UDP | 10.10.10.0/24 | 10.10.99.6 | 514 | Allow syslog viewer from Trusted |
| 6 | Pass | SSH | 10.10.10.10 | 10.10.99.0/24 | 2222 | SSH to management hosts (workstation only) |
| 7 | Block | * | 10.10.10.0/24 | 10.10.20.0/24 | * | No access to Lab from Trusted |
| 8 | Block | * | 10.10.10.0/24 | 10.10.30.0/24 | * | No access to IoT from Trusted |
| 9 | Pass | * | 10.10.10.0/24 | ! RFC1918 | * | Allow all outbound internet |
| 10 | Block | * | * | * | * | Default deny |

### VLAN 20 (Lab) — Outbound & Inter-VLAN

| # | Action | Proto | Source | Dest | Port | Description |
|---|--------|-------|--------|------|------|-------------|
| 1 | Pass | TCP/UDP | 10.10.20.0/24 | 10.10.20.50 | 53 | DNS to local resolver only |
| 2 | Block | TCP/UDP | 10.10.20.0/24 | * | 53 | Block all other DNS (prevent exfil) |
| 3 | Pass | TCP | 10.10.20.0/24 | ! RFC1918 | 80,443 | Allow HTTP/S outbound (tool downloads, updates) |
| 4 | Block | * | 10.10.20.0/24 | 10.10.10.0/24 | * | No access to Trusted |
| 5 | Block | * | 10.10.20.0/24 | 10.10.30.0/24 | * | No access to IoT |
| 6 | Block | * | 10.10.20.0/24 | 10.10.99.0/24 | * | No access to Management |
| 7 | Pass | * | 10.10.20.0/24 | 10.10.20.0/24 | * | Allow intra-VLAN (lab-to-lab) |
| 8 | Block | * | * | * | * | Default deny |

### VLAN 30 (IoT) — Outbound & Inter-VLAN

| # | Action | Proto | Source | Dest | Port | Description |
|---|--------|-------|--------|------|------|-------------|
| 1 | Pass | UDP | 10.10.30.0/24 | 1.1.1.1, 1.0.0.1 | 53 | DNS to Cloudflare only |
| 2 | Pass | UDP | 10.10.30.0/24 | * | 123 | NTP (required by most IoT) |
| 3 | Pass | TCP | 10.10.30.10 | *.meethue.com | 443 | Hue Bridge → cloud |
| 4 | Pass | TCP | 10.10.30.12 | *.ecobee.com | 443 | Ecobee → cloud |
| 5 | Pass | TCP | 10.10.30.11 | * | 80,443 | Roku — needs broad HTTP/S |
| 6 | Pass | TCP | 10.10.30.13–.15 | * | 443 | Kasa plugs → TP-Link cloud |
| 7 | Block | * | 10.10.30.0/24 | RFC1918 | * | Block all access to internal networks |
| 8 | Block | * | * | * | * | Default deny |

### VLAN 99 (Management) — Inbound Only

| # | Action | Proto | Source | Dest | Port | Description |
|---|--------|-------|--------|------|------|-------------|
| 1 | Pass | TCP | 10.10.10.0/24 | 10.10.99.0/24 | 443,3000,8006,8443 | Management UIs from Trusted |
| 2 | Pass | SSH | 10.10.10.10 | 10.10.99.0/24 | 2222 | SSH from workstation only |
| 3 | Pass | UDP | 10.10.99.0/24 | * | 53 | DNS outbound (for updates) |
| 4 | Pass | TCP | 10.10.99.0/24 | ! RFC1918 | 80,443 | HTTP/S outbound (for updates) |
| 5 | Block | * | * | * | * | Default deny |

### WAN — Inbound

| # | Action | Proto | Source | Dest | Port | Description |
|---|--------|-------|--------|------|------|-------------|
| 1 | Pass | UDP | * (US GeoIP only) | WAN address | 1194 | OpenVPN |
| 2 | Pass | UDP | * (US GeoIP only) | WAN address | 51820 | WireGuard |
| 3 | Block | * | * | * | * | Default deny all inbound |

## Components

### Proxmox VE 8.1 (Hypervisor)
The foundation of the lab. Runs all virtualized services including pfSense, monitoring tools, and lab VMs. Chosen over bare-metal installs for snapshot capability, resource isolation, and the ability to spin up/tear down environments quickly.

**Why Proxmox over ESXi:** Free tier includes full clustering, live migration, and ZFS support. ESXi's free license restricts API access and backup automation — both critical for a lab that gets rebuilt frequently.

**Host resource allocation:** With 64GB RAM and 8C/16T, the typical running footprint is ~22GB RAM and 8 vCPUs (pfSense + Kali + all LXCs). That leaves headroom to spin up the AD lab or multiple VulnHub VMs simultaneously without contention.

### pfSense 2.7.2 (Firewall/Router)
Runs as a Proxmox VM with dedicated NIC passthrough (Intel I350 port 1) for WAN. Handles all inter-VLAN routing, NAT, DHCP, and DNS. Unbound is configured as the DNS resolver with DNS over TLS forwarding to Cloudflare (1.1.1.1 / 1.0.0.1) and a blocklist of ~120K ad/tracking domains sourced from the OISD blocklist.

**Key firewall design decisions:**
- Default deny on all VLAN interfaces. Every rule has a logged justification (see rule tables above).
- IoT VLAN has no initiated access to any other VLAN — only responses to established connections from the Management VLAN (for device configuration via the UniFi controller).
- Lab VLAN is fully isolated. DNS is handled by a local Unbound instance (10.10.20.50) within the VLAN to prevent DNS-based data exfiltration during red team exercises.
- SSH uses a non-standard port (2222) and is restricted to a single source IP (10.10.10.10 — my workstation).

### Suricata 7.0 IDS/IPS
Deployed inline on the LAN interface of pfSense. Runs in IPS mode on the IoT and Lab VLANs (active blocking) and IDS mode on the Trusted VLAN (alert only, to avoid disrupting daily use).

**Rule management:**
- ET Open ruleset as baseline (~45,000 rules loaded)
- Custom rules written for lab-specific traffic patterns (e.g., detecting Metasploit reverse shells from VLAN 20, alerting on any VLAN 30 traffic to RFC1918 addresses)
- Suppression list maintained for 23 false-positive signatures from known-good IoT device behavior (Roku health checks, Hue Bridge UPnP discovery)
- Rules reviewed and tuned weekly based on alert volume — currently averaging ~15 legitimate alerts/day after tuning (down from 300+/day at initial deployment)

**Example custom rule:**
```
# Alert on any reverse shell attempt from Lab VLAN
alert tcp 10.10.20.0/24 any -> $EXTERNAL_NET any (msg:"LAB - Possible Reverse Shell Outbound"; flow:established,to_server; content:"/bin/sh"; sid:1000001; rev:1;)
```

### ntopng 6.0 (Flow Monitoring)
Runs as an LXC container (VMID 201) on VLAN 99. Receives mirrored traffic from the pfSense LAN trunk via a dedicated interface. Provides Layer 7 traffic visibility across all VLANs.

**Use cases:**
- Identifying bandwidth anomalies (e.g., an IoT device suddenly uploading 500MB at 3 AM)
- Tracking per-device data consumption on the IoT VLAN
- Verifying VLAN isolation — if ntopng shows any flow between VLAN 30 and VLAN 10, something is misconfigured
- Building the IoT outbound allowlists — each new device goes through a 7-day monitoring period before firewall rules are finalized

**Retention:** Flow data retained for 30 days on the 2TB SATA SSD. Older data is automatically purged.

### VPN Access

**OpenVPN (UDP 1194)** — Primary remote access tunnel into VLAN 10 (Trusted). Certificate-based authentication with 4096-bit RSA keys generated via EasyRSA 3. TLS-auth (ta.key) enabled to prevent DoS on the VPN port by requiring a valid HMAC before the TLS handshake begins. Connected clients receive addresses from 10.10.10.220–.230 and are subject to the same firewall rules as local Trusted devices.

**WireGuard (UDP 51820)** — Secondary tunnel used for quick administrative access to VLAN 99 (Management) from mobile. Uses Curve25519 key pairs with pre-shared keys rotated on the first of each month. Restricted to Proxmox (8006) and pfSense (443) management ports only — no general network access. Assigned client range: 10.10.99.240–.245.

**Key rotation:** VPN credentials are generated and distributed using `scripts/vpn-keygen.sh`, which handles EasyRSA cert generation, client `.ovpn` config templating, WireGuard key pair creation, and CRL updates. Keys are never reused across devices. Revoked certificates are added to the CRL and pushed to pfSense automatically.

## Security Hardening

### Firewall
- Default deny inbound and inter-VLAN on all interfaces
- Outbound filtering per VLAN (IoT restricted to NTP, device-specific cloud endpoints, and DNS to Cloudflare only)
- GeoIP blocking on WAN for inbound connections (allow US only for VPN ports)
- Rate limiting: OpenVPN and WireGuard ports limited to 10 connections/minute per source IP
- Bogon and RFC1918 blocking on WAN interface

### Host Security
- Proxmox root SSH login disabled; admin access via `raine` user with sudo and key-only auth
- SSH hardened: key-only, port 2222, `MaxAuthTries 3`, `ClientAliveInterval 300`, fail2ban active (5 attempts → 1 hour ban)
- Automatic security updates enabled on all Debian-based VMs and LXCs via `unattended-upgrades`
- LUKS full-disk encryption on both Proxmox host drives (unlock via console on boot)
- `PermitRootLogin no` enforced across every VM and container

### Monitoring & Alerting
- Suricata alerts forwarded to rsyslog aggregator at 10.10.99.6 (UDP 514)
- ntopng flow data retained for 30 days
- pfSense system logs and filter logs shipped to rsyslog via syslog-ng
- Daily automated backup of pfSense config XML (`scripts/pfsense-backup.sh` → stored in `/data-zfs/backups/pfsense/`)
- Daily Proxmox VM snapshots at 2 AM via `vzdump` with 7-day retention
- Weekly full backup of all LXC containers to USB drive (manual rotation offsite monthly)

## What I Learned

Building this lab taught me more about networking fundamentals than any coursework. A few specific takeaways:

- **VLAN segmentation is only as good as your firewall rules.** Trunking VLANs to a switch is easy. Writing rules that are restrictive enough to be secure but permissive enough to not break legitimate traffic takes iteration and testing. The IoT VLAN rules took three revisions — the Roku alone required broad HTTP/S access that I initially blocked, breaking all streaming.
- **IDS tuning is a full-time job.** Out-of-the-box Suricata with ET Open rules generated 300+ false positives daily. Getting that down to ~15 real alerts/day took weeks of reviewing logs, writing suppression rules, and learning which signatures are noisy by design. The real skill is understanding which alerts matter and writing suppression rules that don't create blind spots.
- **IoT devices are worse than you think.** During the ntopng monitoring phase, I discovered my smart plugs were making DNS requests to a hardcoded IP (bypassing my resolver), the Roku was doing UPnP discovery broadcasts across subnets, and the thermostat was sending telemetry every 30 seconds. All of this informed the firewall rules.
- **Documentation is infrastructure.** I rebuilt this lab three times before I started documenting configurations. Now every change goes through a documented process, which makes disaster recovery trivial — I restored the full environment from backups in under 45 minutes during a test.

## Future Plans

- [ ] Deploy Wazuh as a SIEM for centralized log correlation and alerting (replacing the basic rsyslog aggregator)
- [ ] Add a DMZ VLAN (VLAN 40 — 10.10.40.0/24) for self-hosted services (Nextcloud, Gitea) with reverse proxy
- [ ] Implement 802.1X port-based authentication via FreeRADIUS on the USW-16-POE
- [ ] Build Terraform configs to manage pfSense rules as code (using the `pfsensible` Ansible collection as a bridge)
- [ ] Set up automated weekly vulnerability scanning of lab VMs with OpenVAS on a cron schedule
- [ ] Add a honeypot on VLAN 20 (Cowrie SSH honeypot) with alerts piped to the SIEM

## Repository Structure

```
├── README.md
├── diagrams/
│   ├── network-topology.drawio        # Editable diagram (draw.io / diagrams.net)
│   └── network-topology.png           # Exported image for README
├── pfsense/
│   ├── firewall-rules.md              # Documented rule table per VLAN (mirrors this README)
│   ├── pfsense-backup.sh              # Automated config XML backup via SSH
│   └── unbound-blocklist.txt          # DNS blocklist (OISD-sourced)
├── suricata/
│   ├── custom-rules/
│   │   └── local.rules                # Lab-specific IDS/IPS signatures
│   ├── suppression.list               # 23 tuned false positive suppressions
│   └── tuning-notes.md                # Rule tuning log with dates and rationale
├── vpn/
│   ├── openvpn-server.conf            # OpenVPN server configuration (sanitized)
│   ├── wireguard-server.conf          # WireGuard interface config (sanitized)
│   └── vpn-keygen.sh                  # Key generation, config templating, CRL management
└── scripts/
    ├── vlan-audit.sh                  # Verify VLAN isolation via cross-VLAN ping/port tests
    ├── backup-all.sh                  # Full lab backup: pfSense config + Proxmox vzdump
    └── iot-monitor.sh                 # Log new IoT device connections and flag unknown MACs
```

> ⚠️ **Note:** All config files in this repo are sanitized — WAN IPs, VPN keys, certificates, and passwords have been redacted or replaced with placeholders.

## Technologies

![pfSense](https://img.shields.io/badge/-pfSense-333333?style=flat&logo=pfsense&logoColor=white)
![Proxmox](https://img.shields.io/badge/-Proxmox-E57000?style=flat&logo=proxmox&logoColor=white)
![Suricata](https://img.shields.io/badge/-Suricata-EF3B2D?style=flat)
![Ubiquiti](https://img.shields.io/badge/-UniFi-0559C9?style=flat&logo=ubiquiti&logoColor=white)
![OpenVPN](https://img.shields.io/badge/-OpenVPN-EA7E20?style=flat&logo=openvpn&logoColor=white)
![WireGuard](https://img.shields.io/badge/-WireGuard-88171A?style=flat&logo=wireguard&logoColor=white)
![ZFS](https://img.shields.io/badge/-ZFS-2A667F?style=flat)
![Unbound](https://img.shields.io/badge/-Unbound%20DNS-333333?style=flat)
