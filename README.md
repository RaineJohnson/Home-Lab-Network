# Secure Home Lab Network Architecture

A production-grade home lab environment built on Proxmox, designed to emulate enterprise network segmentation, threat detection, and secure remote access. This project documents the full architecture, design rationale, and implementation details.

## Architecture Overview

```
                          ┌──────────────┐
                          │   INTERNET   │
                          └──────┬───────┘
                                 │
                          ┌──────┴───────┐
                          │   ISP Modem   │
                          │  (Bridge Mode)│
                          └──────┬───────┘
                                 │
                    ┌────────────┴────────────┐
                    │     pfSense Firewall     │
                    │   (Proxmox VM - 4C/8GB)  │
                    │                          │
                    │  WAN: DHCP from ISP      │
                    │  LAN: 10.0.0.0/16        │
                    │  Suricata IDS/IPS        │
                    │  ntopng Flow Analysis    │
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │   Ubiquiti UniFi Switch  │
                    │   (Managed, 802.1Q)      │
                    └─┬──────┬──────┬────── ┬──┘
                      │      │      │        \  
              ┌───────┴┐ ┌───┴───┐ ┌┴──────=┐ ┌──────────┐
              │VLAN 10 │ │VLAN 20│ │VLAN 30 │ │ VLAN 99  │
              │TRUSTED │ │  LAB  │ │  IOT   │ │MANAGEMENT│
              │10.0.10 │ │10.0.20│ │10.0.30 │ │ 10.0.99  │
              │  /24   │ │  /24  │ │  /24   │ │   /24    │
              └────────┘ └───────┘ └────────┘ └──────────┘
```

## Network Segments

### VLAN 10 — Trusted (10.0.10.0/24)
Primary workstations and personal devices. Full outbound internet access with DNS filtering. This is the only VLAN permitted to initiate SSH connections to VLAN 99 (management).

### VLAN 20 — Lab (10.0.20.0/24)
Security testing and development environment. Isolated from trusted and IoT segments. Runs vulnerable VMs (Metasploitable, DVWA) for offensive security practice. Outbound internet is filtered and logged — no direct access to other VLANs.

### VLAN 30 — IoT (10.0.30.0/24)
Isolated network for IoT devices. No inter-VLAN routing permitted. Outbound access restricted to specific ports and destinations. All traffic monitored by Suricata with IoT-specific rulesets.

### VLAN 99 — Management (10.0.99.0/24)
Administrative access only. Hosts Proxmox management interface, pfSense admin panel, switch management, and UniFi controller. Accessible only from VLAN 10 via explicit firewall rules.

## Components

### Proxmox VE (Hypervisor)
The foundation of the lab. Runs all virtualized services including pfSense, monitoring tools, and lab VMs. Chosen over bare-metal installs for snapshot capability, resource isolation, and the ability to spin up/tear down environments quickly.

**Why Proxmox over ESXi:** Free tier includes full clustering, live migration, and ZFS support. ESXi's free license restricts API access and backup automation — both critical for a lab that gets rebuilt frequently.

### pfSense (Firewall/Router)
Runs as a Proxmox VM with dedicated NIC passthrough for WAN. Handles all inter-VLAN routing, NAT, DHCP, and DNS (with DNS over TLS to Cloudflare). Firewall rules follow a default-deny model — every permitted flow is explicitly documented.

**Key firewall design decisions:**
- Default deny on all VLAN interfaces. Every rule has a logged justification.
- IoT VLAN has no initiated access to any other VLAN — only responses to established connections from the management VLAN (for device configuration).
- Lab VLAN is fully isolated. Even DNS is handled by a local resolver within the VLAN to prevent DNS-based data exfiltration during red team exercises.

### Suricata IDS/IPS
Deployed inline on the LAN interface of pfSense. Runs in IPS mode on the IoT and Lab VLANs (active blocking) and IDS mode on the Trusted VLAN (alert only, to avoid disrupting daily use).

**Rule management:**
- ET Open ruleset as baseline
- Custom rules written for lab-specific traffic patterns (e.g., detecting Metasploit reverse shells from VLAN 20)
- Suppression list maintained to reduce false positives from known-good IoT device behavior
- Rules reviewed and tuned weekly based on alert volume

### ntopng (Flow Monitoring)
Provides Layer 7 traffic visibility across all VLANs. Used to identify bandwidth anomalies, track per-device data consumption on the IoT VLAN, and verify that VLAN isolation is enforced (i.e., no cross-VLAN flows where none should exist).

### VPN Access

**OpenVPN** — Primary remote access tunnel into VLAN 10 (Trusted). Certificate-based authentication with 4096-bit RSA keys. TLS-auth enabled to prevent DoS on the VPN port. Connected clients receive VLAN 10 addresses and are subject to the same firewall rules as local trusted devices.

**WireGuard** — Secondary tunnel used for quick administrative access to VLAN 99 (Management) from mobile. Pre-shared keys rotated monthly. Restricted to Proxmox and pfSense management ports only — no general network access.

**Key rotation:** VPN credentials are generated and distributed using a custom Bash script that handles key generation, client config templating, and revocation list updates. Keys are never reused across devices.

## Security Hardening

### Firewall
- Default deny inbound and inter-VLAN
- Outbound filtering per VLAN (IoT restricted to NTP, MQTT broker, and manufacturer update servers only)
- GeoIP blocking on WAN for inbound connections (allow US only for VPN)
- Rate limiting on VPN and SSH ports

### Host Security
- Proxmox root login disabled; admin access via sudo-enabled user only
- SSH hardened: key-only auth, non-standard port, fail2ban active
- Automatic security updates enabled on all Debian-based VMs
- LUKS encryption on Proxmox host drives

### Monitoring & Alerting
- Suricata alerts forwarded to a local syslog aggregator
- ntopng flow data retained for 30 days
- pfSense system logs shipped to centralized logging
- Daily automated backup of pfSense config and Proxmox VM snapshots

## What I Learned

Building this lab taught me more about networking fundamentals than any coursework. A few specific takeaways:

- **VLAN segmentation is only as good as your firewall rules.** Trunking VLANs to a switch is easy. Writing rules that are restrictive enough to be secure but permissive enough to not break legitimate traffic takes iteration and testing.
- **IDS tuning is a full-time job.** Out-of-the-box Suricata with ET Open rules generated hundreds of false positives daily. The real skill is understanding which alerts matter and writing suppression rules that don't create blind spots.
- **Documentation is infrastructure.** I rebuilt this lab three times before I started documenting configurations. Now every change goes through a documented process, which makes disaster recovery trivial.

## Future Plans

- [ ] Deploy a SIEM stack (Wazuh or Security Onion) for centralized log correlation
- [ ] Add a DMZ VLAN for self-hosted services (Nextcloud, Gitea)
- [ ] Implement 802.1X port-based authentication on the managed switch
- [ ] Build a Terraform provider for pfSense to manage rules as code
- [ ] Set up automated vulnerability scanning of lab VMs with OpenVAS on a cron schedule

## Repository Structure

```
├── README.md
├── diagrams/
│   ├── network-topology.drawio        # Editable diagram (draw.io)
│   └── network-topology.png           # Exported image
├── pfsense/
│   ├── firewall-rules.md              # Documented rule table per VLAN
│   └── config-backup.sh               # Automated config backup script
├── suricata/
│   ├── custom-rules/                  # Custom IDS/IPS signatures
│   ├── suppression.list               # Tuned false positive suppressions
│   └── tuning-notes.md                # Rule tuning decisions and rationale
├── vpn/
│   ├── openvpn-server.conf            # OpenVPN server configuration
│   ├── wireguard-server.conf          # WireGuard server configuration
│   └── key-rotation.sh                # Automated key management script
└── scripts/
    ├── vlan-audit.sh                  # Verify VLAN isolation
    └── backup-all.sh                  # Full lab backup automation
```

## Technologies

![pfSense](https://img.shields.io/badge/-pfSense-333333?style=flat&logo=pfsense&logoColor=white)
![Proxmox](https://img.shields.io/badge/-Proxmox-E57000?style=flat&logo=proxmox&logoColor=white)
![Suricata](https://img.shields.io/badge/-Suricata-EF3B2D?style=flat)
![Ubiquiti](https://img.shields.io/badge/-UniFi-0559C9?style=flat&logo=ubiquiti&logoColor=white)
![OpenVPN](https://img.shields.io/badge/-OpenVPN-EA7E20?style=flat&logo=openvpn&logoColor=white)
![WireGuard](https://img.shields.io/badge/-WireGuard-88171A?style=flat&logo=wireguard&logoColor=white)
