# 🏠 Homelab Infrastructure

> A production-grade home network and virtualization environment built to practice enterprise security, network segmentation, and infrastructure operations.

[![pfSense](https://img.shields.io/badge/Firewall-pfSense-blue?logo=pfsense)](https://www.pfsense.org/)
[![Wazuh](https://img.shields.io/badge/SIEM-Wazuh-orange)](https://wazuh.com/)
[![Suricata](https://img.shields.io/badge/IDS%2FIPS-Suricata-red)](https://suricata.io/)
[![Windows Server](https://img.shields.io/badge/Hypervisor-Windows%20Server-0078D6?logo=windows)](https://www.microsoft.com/en-us/windows-server)
[![OpenVPN](https://img.shields.io/badge/VPN-OpenVPN-orange?logo=openvpn)](https://openvpn.net/)

---

## Architecture Overview

```
Internet ──── Gateway Router
                    │
             ┌──────▼──────┐
             │  WAN NIC    │
             └──────┬──────┘
                    │
        ┌───────────▼─────────────────────────────┐
        │          Windows Server (Hypervisor)    │
        │                                         │
        │  ┌─────────────────────────────────┐    │
        │  │         pfSense VM              │    │
        │  │  ┌──────┐ ┌──────┐ ┌──────┐     │    │
        │  │  │ WAN  │ │ LAN  │ │ MGMT │     │    │
        │  │  └──┬───┘ └──┬───┘ └──┬───┘     │    │
        │  └─────┼─────────┼────────┼────────┘    │
        │        │  vSwitches        │            │
        │  ┌─────▼──┐  ┌───▼──┐  ┌──▼────┐        │
        │  │vSW:WAN │  │vSW:  │  │vSW:   │        │
        │  └────────┘  │ LAN  │  │ MGMT  │        │
        │              └──┬───┘  └──┬────┘        │
        │            ┌────┴──┐  ┌───┴───┐         │
        │            │Rustdesk│  │Bastion│        │
        │            │Web Srv │  │Wazuh  │        │
        │            └───────┘  └───────┘         │
        └─────────────────────────────────────────┘
                    │           │         │
             VLAN 10:LAN  VLAN 99:MGMT  VLAN 20:IoT
                    │           │         │
              Managed Switch ───┴─────────┘
                    │
             802.1Q Trunk
                    │
              Access Point
               /        \
        SSID: LAN     SSID: IoT
```

---

## Network Segmentation

| Network | Subnet | VLAN | Purpose |
|---------|--------|------|---------|
| LAN | `192.168.1.0/24` | VLAN 10 | Trusted workstations & services |
| IoT | `192.168.20.0/24` | VLAN 20 | Isolated smart home devices |
| Management | `172.16.99.0/28` | VLAN 99 | Infrastructure management only |

**Firewall rules enforce strict inter-VLAN isolation** — IoT devices have no path to LAN or MGMT, and the management network is only reachable through the bastion host.

---

## Virtual Machines

| VM | Role | Network | Description |
|----|------|---------|-------------|
| **pfSense** | Firewall / Router | WAN, LAN, MGMT, IoT | Central routing and security enforcement |
| **Rustdesk** | Remote Access | LAN | Self-hosted remote desktop relay server |
| **Web Server** | Hosting | LAN | Internal web services |
| **Bastion Host** | Jump Server | MGMT | Single ingress point for management network |
| **Wazuh** | SIEM | MGMT | Security event collection and alerting |

---

## Security Stack

### pfSense
- **Suricata IDS/IPS** — Inline traffic inspection with rule sets for network threat detection
- **pfBlockerNG** — DNS-based ad/malware blocking with curated threat intelligence feeds
- **OpenVPN** — Remote access VPN with certificate-based authentication
- **Dynamic DNS (NoIP)** — Automatic WAN IP tracking for VPN endpoint

### Wazuh SIEM
- Aggregates logs from pfSense (Suricata alerts, firewall events, OpenVPN connections)
- **LDAP authentication** for centralized user login management
- **Slack integration** — Real-time security alerts forwarded to a dedicated Slack channel
- Correlation rules for detecting lateral movement, brute force, and policy violations

### Access Control
- Bastion host is the **only** path into the management VLAN
- Remote access via self-hosted **Rustdesk** (no third-party relay dependency)
- Management VLAN subnet is a `/28` — minimal blast radius by design

---

## Physical Infrastructure

```
┌─────────────────────────────────────┐
│           Windows Server            │
│  NIC1: WAN  │  NIC2: LAN            │
│  NIC3: MGMT │  NIC4: IoT            │
└──────────────┬──────────────────────┘
               │ Tagged VLANs (802.1Q)
        ┌──────▼───────┐
        │ Managed      │
        │ Switch       │
        │ VLAN 10/20/99│
        └──────┬───────┘
               │ 802.1Q Trunk
        ┌──────▼────────┐
        │  Access Point │
        │  SSID: LAN    │
        │  SSID: IoT    │
        └───────────────┘
```

- **4 physical NICs** mapped 1:1 to virtual switches in Hyper-V
- **Managed switch** enforces VLAN boundaries at layer 2
- **Access point** carries multiple SSIDs, each trunked to the correct VLAN

---

*This lab is intentionally designed to mirror enterprise security architecture at a smaller scale — segmented networks, centralized logging, and a defense-in-depth approach rather than a flat trusted network.*