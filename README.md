# 🔬 SOC Home Lab

![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-VirtualBox-183A61?style=flat-square&logo=virtualbox)
![SIEM](https://img.shields.io/badge/SIEM-Splunk-FF5733?style=flat-square)
![Framework](https://img.shields.io/badge/Framework-MITRE%20ATT%26CK-B22222?style=flat-square)

A fully virtualized Security Operations Center environment built on VirtualBox. The lab spans six isolated network segments covering enterprise network simulation, Active Directory attack/defense, malware analysis, DFIR, and SIEM operations — all routed through a pfSense firewall.

---

## 🗺️ Network Topology

```
                         ┌──────────────────────────────────┐
                         │        pfSense Firewall           │
                         │     Gateway + Segmentation        │
                         │         vtnet0 = WAN (NAT)        │
                         └────┬──────┬──────┬──────┬────┬───┘
                              │      │      │      │    │
             ┌────────────────┘      │      │      │    └──────────────────┐
             │                       │      │      │                       │
   ┌─────────▼──────┐   ┌────────────▼──┐  ┌──────▼──────┐   ┌───────────▼────────┐
   │  LAN (vtnet1)  │   │CYBER_RANGE    │  │  AD_LAB     │   │  ISOLATED          │
   │  10.0.0.0/24   │   │(vtnet2)       │  │  (vtnet3)   │   │  (vtnet4)          │
   │                │   │10.6.6.0/24    │  │ 10.80.80.0/24│  │  10.99.99.0/24     │
   │ Kali Linux     │   │               │  │              │   │                    │
   │ 10.0.0.2 (Mgmt)│   │ Metasploitable│  │ Win Server   │   │ FLARE VM           │
   │                │   │ Chronos       │  │ 2019 (DC)    │   │ REMnux             │
   └────────────────┘   │ (CTF VMs)     │  │ Win 10 x2    │   └────────────────────┘
                        └───────────────┘  └─────────────┘
                                                               ┌────────────────────┐
                                                               │  SECURITY (vtnet5) │
                                                               │  10.10.10.0/24     │
                                                               │                    │
                                                               │ Tsurugi (DFIR)     │
                                                               │ 10.10.10.2         │
                                                               │ Ubuntu/Splunk      │
                                                               │ 10.10.10.13        │
                                                               └────────────────────┘
```

---

## 📦 Network Segments

| Interface | Name | Subnet | Purpose |
|---|---|---|---|
| vtnet0 | WAN | NAT (VirtualBox) | Internet access via host |
| vtnet1 | LAN | 10.0.0.0/24 | Management — Kali Linux |
| vtnet2 | CYBER_RANGE | 10.6.6.0/24 | Vulnerable VMs for CTF/attack practice |
| vtnet3 | AD_LAB | 10.80.80.0/24 | Active Directory domain environment |
| vtnet4 | ISOLATED | 10.99.99.0/24 | Air-gapped malware analysis lab |
| vtnet5 | SECURITY | 10.10.10.0/24 | DFIR and SIEM tools |

---

## 🖥️ Virtual Machines

### Management
| VM | OS | IP | Role |
|---|---|---|---|
| pfSense | pfSense CE 2.7.2 | 10.0.0.1 (gateway) | Firewall, router, DHCP/DNS |
| Kali Linux | Kali 2023.4 | 10.0.0.2 (static) | Management, attack VM, pfSense admin |

### Cyber Range (CYBER_RANGE — 10.6.6.0/24)
| VM | IP | Purpose |
|---|---|---|
| Metasploitable 2 | 10.6.6.12 | Intentionally vulnerable Linux target |
| Chronos | 10.6.6.13 | CTF-style vulnerable VM |

### Active Directory Lab (AD_LAB — 10.80.80.0/24)
| VM | OS | IP | Role |
|---|---|---|---|
| DC1 | Windows Server 2019 | 10.80.80.2 (static) | Domain Controller, DNS, DHCP, CA |
| Win10-User1 | Windows 10 Enterprise | DHCP (10.80.80.11+) | Domain client — John |
| Win10-User2 | Windows 10 Enterprise | DHCP (10.80.80.11+) | Domain client — Jane |

**Domain:** `ad.lab`

### Malware Analysis Lab (ISOLATED — 10.99.99.0/24)
| VM | OS | IP | Purpose |
|---|---|---|---|
| FLARE VM | Windows 10 Enterprise | 10.99.99.11 | Windows malware analysis |
| REMnux | REMnux 7 | 10.99.99.12 | Linux malware analysis |

### Security (SECURITY — 10.10.10.0/24)
| VM | OS | IP | Purpose |
|---|---|---|---|
| Tsurugi Linux | Tsurugi 2023.2 | 10.10.10.2 (static) | DFIR — forensics and IR tools |
| Ubuntu/Splunk | Ubuntu 22.04 LTS | 10.10.10.13 | SIEM — Splunk Enterprise 9.x |

---

## 🔥 pfSense Firewall Rules Summary

### LAN (10.0.0.0/24)
| Action | Source | Destination | Notes |
|---|---|---|---|
| Block | LAN subnets | WAN subnets | Prevent access to VirtualBox host services |
| Allow | LAN subnets | Any | Full outbound access for management |

### CYBER_RANGE (10.6.6.0/24)
| Action | Source | Destination | Notes |
|---|---|---|---|
| Allow | CYBER_RANGE | CYBER_RANGE | Internal segment traffic |
| Allow | CYBER_RANGE | 10.0.0.2 | Access to Kali Linux only |
| Allow | CYBER_RANGE | Non-RFC1918 | Internet access for CTF challenges |
| Block | CYBER_RANGE | Any | Default deny everything else |

### AD_LAB (10.80.80.0/24)
| Action | Source | Destination | Notes |
|---|---|---|---|
| Block | AD_LAB | WAN subnets | No access to host services |
| Block | AD_LAB | CYBER_RANGE | Prevent cross-contamination |
| Allow | AD_LAB | Any | Access to LAN, SECURITY, internet |

### ISOLATED (10.99.99.0/24)
| Action | Source | Destination | Notes |
|---|---|---|---|
| Allow | ISOLATED | 10.10.10.2 port 22 | SSH to Tsurugi (file transfer only) |
| Block | ISOLATED | Any | Full network isolation — no internet |

### SECURITY (10.10.10.0/24)
| Action | Source | Destination | Notes |
|---|---|---|---|
| Block | SECURITY | WAN subnets | No access to host services |
| Block | SECURITY | LAN subnets | Isolated from management network |
| Allow | SECURITY | Any | Access to AD_LAB, ISOLATED, internet |

---

## 🎯 Active Directory — Attack Scenarios

| # | Scenario | MITRE Tactic | Technique ID |
|---|---|---|---|
| 1 | Password Spraying via SMB | Credential Access | [T1110.003](https://attack.mitre.org/techniques/T1110/003/) |
| 2 | GPO Privilege Escalation | Privilege Escalation | [T1484.001](https://attack.mitre.org/techniques/T1484/001/) |
| 3 | Pass-the-Hash Lateral Movement | Lateral Movement | [T1550.002](https://attack.mitre.org/techniques/T1550/002/) |
| 4 | Scheduled Task Persistence | Persistence | [T1053.005](https://attack.mitre.org/techniques/T1053/005/) |
| 5 | Kerberoasting | Credential Access | [T1558.003](https://attack.mitre.org/techniques/T1558/003/) |

The AD environment is intentionally made exploitable using the [vulnerable-AD-plus](https://github.com/WaterExecution/vulnerable-AD-plus) script. GPOs are configured to disable Windows Defender, enable WinRM, RDP, and RPC across all domain machines.

---

## 🦠 Malware Analysis Workflow

Files are transferred to the air-gapped ISOLATED subnet using SCP from Tsurugi Linux (SECURITY subnet). Tsurugi is the only machine with SSH access to ISOLATED per firewall rules.

```
Internet → Tsurugi Linux (10.10.10.2) → SCP → FLARE VM / REMnux (10.99.99.x)
```

See: [`/malware-analysis/file-transfer-workflow.md`](./malware-analysis/file-transfer-workflow.md)

---

## 📁 Repository Structure

```
soc-home-lab/
├── README.md
├── network/
│   ├── pfsense-config.md          # Full pfSense setup and firewall rules
│   └── network-diagram.md         # Topology documentation
├── active-directory/
│   ├── ad-setup.md                # DC, DHCP, DNS, GPO configuration
│   └── attack-simulations.md      # AD attack scenarios and detections
├── malware-analysis/
│   ├── flare-vm-setup.md          # FLARE VM build and configuration
│   ├── remnux-setup.md            # REMnux setup notes
│   └── file-transfer-workflow.md  # SCP workflow from SECURITY to ISOLATED
├── dfir/
│   └── tsurugi-setup.md           # Tsurugi Linux DFIR environment
├── splunk/
│   ├── splunk-setup.md            # Splunk install and Universal Forwarder config
│   └── dashboards.md              # Detection dashboards
└── docs/
    └── incident-reports/          # Structured IR reports per scenario
```

---

## 🧰 Full Tools & Software Inventory

| Category | Tool | Version | Segment |
|---|---|---|---|
| Hypervisor | Oracle VirtualBox | 7.x | Host |
| Firewall | pfSense CE | 2.7.2 | WAN/All |
| Management/Attack | Kali Linux | 2023.4 | LAN |
| CTF Target | Metasploitable 2 | — | CYBER_RANGE |
| CTF Target | Chronos | — | CYBER_RANGE |
| Domain Controller | Windows Server 2019 | — | AD_LAB |
| AD Clients | Windows 10 Enterprise | 22H2 | AD_LAB |
| Windows Malware Analysis | FLARE VM | Latest | ISOLATED |
| Linux Malware Analysis | REMnux | 7 | ISOLATED |
| DFIR | Tsurugi Linux | 2023.2 | SECURITY |
| SIEM | Splunk Enterprise | 9.x | SECURITY |
| SIEM Host | Ubuntu | 22.04 LTS | SECURITY |

---

## 🔗 Related Repositories

- [splunk-detection-rules](https://github.com/Qwortie/splunk-detection-rules) — SPL detection rules used in this lab
- [malware-analysis-reports](https://github.com/Qwortie/malware-analysis-reports) — Write-ups from the ISOLATED malware lab
