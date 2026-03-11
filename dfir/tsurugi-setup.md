# Tsurugi Linux — DFIR Setup

## Overview

Tsurugi Linux is a DFIR-focused OS pre-loaded with forensics and incident response tools. It sits on the SECURITY segment alongside Splunk and serves as the gateway into the air-gapped ISOLATED malware analysis lab via SCP.

| Setting | Value |
|---|---|
| OS | Tsurugi Linux 2023.2 |
| Segment | SECURITY (10.10.10.0/24) |
| IP | 10.10.10.2 (static via pfSense DHCP reservation) |
| Role | DFIR analysis, malware sample staging, SCP bridge to ISOLATED |

---

## VM Specs

| Resource | Value |
|---|---|
| RAM | 4096 MB |
| Disk | 150 GB (minimum — Tsurugi install requires 110 GB+) |
| Network | Internal Network — LAN 4 (SECURITY segment) |
| VirtualBox EFI | Enabled (required for Tsurugi 2024.1+) |

---

## Key Tools Available

**Digital Forensics**
- Autopsy, The Sleuth Kit, bulk_extractor
- Volatility (memory forensics)
- Foremost, Scalpel (file carving)
- ExifTool, binwalk

**Network Analysis**
- Wireshark, tcpdump, NetworkMiner
- Zeek (formerly Bro)

**Malware Analysis (light)**
- YARA, ClamAV
- `strings`, `file`, `hexdump`
- olevba, pdfid (document analysis)

**IR & Response**
- Log2timeline / Plaso (timeline analysis)
- RegRipper (Windows registry analysis)
- Chainsaw (Windows event log hunting)

---

## Role in the Lab

### 1. Malware Sample Staging
Tsurugi downloads malware samples from public sources (MalwareBazaar, Any.run) over the internet, then transfers them into the ISOLATED lab via SCP. It is the **only machine with SSH access to ISOLATED** per pfSense firewall rules.

```bash
# Download sample
wget "https://mb-api.abuse.ch/api/v1/" --post-data "query=get_file&sha256_hash=<HASH>" -O sample.zip

# Transfer to FLARE VM
scp sample.zip david@10.99.99.11:/C:/Users/David/Downloads/

# Transfer to REMnux
scp remnux@10.99.99.12:~/Downloads/
```

### 2. Memory & Disk Forensics
When a VM in the lab is suspected to be compromised, memory dumps and disk images can be analyzed on Tsurugi using Volatility and Autopsy without risk of further contamination.

### 3. Log Timeline Analysis
Windows event logs and pfSense firewall logs exported from Splunk can be processed through Plaso/Log2timeline on Tsurugi to build attack timelines.

---

## Static IP Configuration

Static IP is assigned via pfSense DHCP reservation rather than hardcoded on the VM, ensuring it survives network adapter resets.

**To refresh the IP after pfSense assigns the reservation:**
```bash
sudo ip l set enp0s3 down && sudo ip l set enp0s3 up
ip a enp0s3  # Verify 10.10.10.2
```

---

## SSH Configuration

SSH is installed but **kept disabled when not in use** to minimize the attack surface.

```bash
# Enable (when transferring files)
sudo systemctl start ssh

# Disable (after transfer complete)
sudo systemctl stop ssh

# Check status
systemctl status ssh
```
