# pfSense Firewall Configuration

## Overview

pfSense CE 2.7.2 runs as the gateway and firewall for the entire lab. It is always the **first VM booted** and the **last VM shut down**. All inter-segment traffic is routed through pfSense, and each segment has purpose-built firewall rules enforcing isolation.

pfSense is managed via its Web UI from the Kali Linux VM at `https://10.0.0.1`.

---

## Network Interfaces

| Interface | vtnet | Subnet | Gateway IP | DHCP Range | Notes |
|---|---|---|---|---|---|
| WAN | vtnet0 | NAT (VirtualBox) | Assigned by VirtualBox | — | Internet via host system |
| LAN | vtnet1 | 10.0.0.0/24 | 10.0.0.1 | 10.0.0.11–10.0.0.243 | Management segment |
| CYBER_RANGE | vtnet2 | 10.6.6.0/24 | 10.6.6.1 | 10.6.6.11–10.6.6.243 | Vulnerable VMs |
| AD_LAB | vtnet3 | 10.80.80.0/24 | 10.80.80.1 | DC-managed (10.80.80.11–253) | DHCP handled by Windows DC |
| ISOLATED | vtnet4 | 10.99.99.0/24 | 10.99.99.1 | 10.99.99.11–10.99.99.243 | Air-gapped malware lab |
| SECURITY | vtnet5 | 10.10.10.0/24 | 10.10.10.1 | 10.10.10.11–10.10.10.243 | DFIR + SIEM |

> **Note:** VirtualBox GUI supports only 4 network interfaces. vtnet4 (ISOLATED) and vtnet5 (SECURITY) were added via VBoxManage CLI.

```powershell
# Adding ISOLATED interface (Adapter 5)
VBoxManage modifyvm "pfSense" --nic5 intnet
VBoxManage modifyvm "pfSense" --nictype5 virtio
VBoxManage modifyvm "pfSense" --intnet5 "LAN 3"
VBoxManage modifyvm "pfSense" --cableconnected5 on

# Adding SECURITY interface (Adapter 6)
VBoxManage modifyvm "pfSense" --nic6 intnet
VBoxManage modifyvm "pfSense" --nictype6 virtio
VBoxManage modifyvm "pfSense" --intnet6 "LAN 4"
VBoxManage modifyvm "pfSense" --cableconnected6 on
```

---

## Static IP Assignments

| VM | IP | Segment |
|---|---|---|
| Kali Linux (management) | 10.0.0.2 | LAN |
| Windows Server 2019 (DC) | 10.80.80.2 | AD_LAB (manual static) |
| Tsurugi Linux (DFIR) | 10.10.10.2 | SECURITY |

Static IPs for Kali and Tsurugi are configured via pfSense **DHCP Leases** (Status → DHCP Leases → assign static mapping).

The DC's static IP is set manually on the Windows network adapter — AD_LAB has pfSense DHCP disabled because the DC runs its own DHCP server for the domain.

---

## Firewall Rules

### LAN Rules (10.0.0.0/24)

| Priority | Action | Protocol | Source | Destination | Description |
|---|---|---|---|---|---|
| 1 | Block | IPv4+IPv6/Any | LAN subnets | WAN subnets | Block access to VirtualBox host services |
| 2 | Allow | IPv4+IPv6/Any | LAN subnets | Any | Full outbound — management needs unrestricted access |

### CYBER_RANGE Rules (10.6.6.0/24)

Uses a `RFC1918` alias covering all private IP space:
- 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 169.254.0.0/16, 127.0.0.0/8

| Priority | Action | Protocol | Source | Destination | Description |
|---|---|---|---|---|---|
| 1 | Allow | Any | CYBER_RANGE subnets | CYBER_RANGE address | Internal segment traffic |
| 2 | Allow | Any | CYBER_RANGE subnets | 10.0.0.2 | Access to Kali Linux only |
| 3 | Allow | Any | CYBER_RANGE subnets | NOT RFC1918 (inverted) | Internet access — needed for some CTF challenges |
| 4 | Block | IPv4+IPv6/Any | CYBER_RANGE subnets | Any | Default deny everything else |

### AD_LAB Rules (10.80.80.0/24)

| Priority | Action | Protocol | Source | Destination | Description |
|---|---|---|---|---|---|
| 1 | Block | IPv4+IPv6/Any | AD_LAB subnets | WAN subnets | Block access to VirtualBox host services |
| 2 | Block | IPv4+IPv6/Any | AD_LAB subnets | CYBER_RANGE subnets | Prevent AD machines reaching vulnerable VMs |
| 3 | Allow | IPv4+IPv6/Any | AD_LAB subnets | Any | Allow access to SECURITY, internet, and other subnets |

### ISOLATED Rules (10.99.99.0/24)

| Priority | Action | Protocol | Source | Destination | Description |
|---|---|---|---|---|---|
| 1 | Allow | TCP | ISOLATED subnets | 10.10.10.2 port 22 | SSH to Tsurugi for file transfer only |
| 2 | Block | IPv4+IPv6/Any | ISOLATED subnets | Any | Full network block — malware cannot reach internet or other segments |

### SECURITY Rules (10.10.10.0/24)

| Priority | Action | Protocol | Source | Destination | Description |
|---|---|---|---|---|---|
| 1 | Block | IPv4+IPv6/Any | SECURITY subnets | WAN subnets | Block access to VirtualBox host services |
| 2 | Block | IPv4+IPv6/Any | SECURITY subnets | LAN subnets | Isolate from management network |
| 3 | Allow | IPv4+IPv6/Any | SECURITY subnets | Any | Allow access to AD_LAB (for Splunk forwarder), ISOLATED (for SCP), internet |

---

## Key Design Decisions

**Why is AD_LAB DHCP disabled on pfSense?**
The Windows Server 2019 Domain Controller runs its own DHCP server for the `ad.lab` domain. If pfSense also ran DHCP on this interface there would be two competing DHCP servers. The DC's DHCP is required so that domain-joined clients receive proper DNS pointing to the DC.

**Why does ISOLATED only allow SSH to Tsurugi?**
The malware analysis VMs (FLARE VM, REMnux) have zero internet access to prevent malware from phoning home or spreading. The single SSH exception to Tsurugi Linux (10.10.10.2) is the only controlled pathway to get malware samples into the lab — downloaded on Tsurugi, then SCP'd across.

**Why does SECURITY block the LAN segment?**
The Kali Linux management VM lives on LAN. Blocking SECURITY → LAN prevents a compromised analysis machine from pivoting to the management network.

**Why is the WAN RFC1918 block disabled?**
pfSense's default setting blocks private IP ranges on the WAN interface assuming WAN uses a real public IP. Since our WAN is a VirtualBox NAT adapter using a private IP address, this default must be unchecked or internet access breaks entirely.

---

## DNS Configuration

- **pfSense DNS Resolver** handles DNS for LAN, CYBER_RANGE, ISOLATED, and SECURITY segments
- **Windows Server 2019 (DC)** handles DNS for AD_LAB with a forwarder pointing to pfSense at `10.80.80.1`
- pfSense DNS Resolver has `DNSSEC`, `DNS over TLS`, and `Forwarding Mode` enabled

---

## Lab Startup Order

1. **pfSense** — must be first; all other VMs depend on it for routing
2. **Kali Linux** — used to manage pfSense via web UI
3. **Windows Server 2019 (DC)** — must be up before AD clients
4. All other VMs in any order

**Shutdown order is the reverse:** all VMs first, pfSense last.
