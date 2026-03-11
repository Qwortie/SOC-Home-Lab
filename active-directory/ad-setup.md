# Active Directory Lab Setup

## Domain Overview

| Setting | Value |
|---|---|
| Domain Name | ad.lab |
| NetBIOS Name | AD |
| Domain Controller | DC1 — Windows Server 2019 |
| DC IP (static) | 10.80.80.2 |
| Default Gateway | 10.80.80.1 (pfSense AD_LAB interface) |
| DNS Server | 10.80.80.2 (DC handles DNS; forwards to pfSense at 10.80.80.1) |
| DHCP | Managed by DC — range 10.80.80.11–10.80.80.253 |
| Client 1 | Win10-User1 (John) |
| Client 2 | Win10-User2 (Jane) |

---

## Domain Controller Configuration

### Services Installed
- Active Directory Domain Services (AD DS)
- DNS Server (with forwarder → 10.80.80.1)
- DHCP Server (scope: 10.80.80.11–10.80.80.253, lease 365 days)
- Active Directory Certificate Services (CA)

### Network Adapter (manual static — pfSense DHCP disabled on AD_LAB)
```
IP address:          10.80.80.2
Subnet mask:         255.255.255.0
Default gateway:     10.80.80.1
Preferred DNS:       10.80.80.2  (points to itself)
```

### DNS Forwarder
DNS queries the DC cannot resolve are forwarded to pfSense at `10.80.80.1`, which handles upstream resolution via the VirtualBox NAT.

---

## User Accounts

| Username | Type | Group | VM |
|---|---|---|---|
| Administrator | Domain Admin | Domain Admins | DC1 |
| [domain-admin] | Custom Admin | Domain Admins | DC1 |
| John Doe | Standard User | Domain Users | Win10-User1 |
| Jane Doe | Standard User | Domain Users | Win10-User2 |

---

## Making the Lab Exploitable

The AD environment is intentionally misconfigured for attack simulation using the [vulnerable-AD-plus](https://github.com/WaterExecution/vulnerable-AD-plus) script.

```powershell
# Run on DC as Administrator
Set-ExecutionPolicy -ExecutionPolicy Bypass -Force

[System.Net.WebClient]::new().DownloadString('https://raw.githubusercontent.com/WaterExecution/vulnerable-AD-plus/master/vulnadplus.ps1') -replace 'change\.me', 'ad.lab' | Invoke-Expression
```

---

## Group Policy Objects

### Disable Protections
Disables Windows Defender and Windows Firewall across all domain machines to allow attack tools to run unobstructed.

- `Computer Config → Admin Templates → Windows Components → Windows Defender Antivirus` → **Enabled (disabled)**
- `Computer Config → Admin Templates → Windows Components → Windows Defender Antivirus → Real-time Protection` → **Enabled (disabled)**
- `Computer Config → Admin Templates → Network → Windows Defender Firewall → Domain Profile` → **Disabled**

### Local Admin Remote Login
Enables local admin accounts to authenticate over the network (removes UAC token filtering) — required for pass-the-hash and lateral movement techniques.

```
Registry: HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
Value:     LocalAccountTokenFilterPolicy = 1 (REG_DWORD)
```

### Enable WinRM Server
Enables Windows Remote Management across all domain machines — allows remote command execution and lateral movement.

- Allow remote server management through WinRM → **Enabled** (IPv4 filter: `*`)
- Allow Basic authentication → **Enabled**
- Allow unencrypted traffic → **Enabled**
- Windows Remote Management (WS-Management) service → **Automatic / Start**
- Allow Remote Shell Access → **Enabled**

### Enable RDP
Enables Remote Desktop across all domain machines.

- `Computer Config → Admin Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → Connections` → **Enabled**

### Enable RPC
Enables Remote Procedure Call endpoint mapper client authentication.

- `Computer Config → Admin Templates → System → Remote Procedure Call → Enable RPC Endpoint Mapper Client Authentication` → **Enabled**

### Applying Policies
```powershell
gpupdate /force
```

---

## Attack Simulations

### 1. Password Spraying (T1110.003)

**Tool:** CrackMapExec from Kali Linux (LAN segment, routed via pfSense to AD_LAB)

```bash
crackmapexec smb 10.80.80.2 -u users.txt -p 'Winter2024!' --continue-on-success
```

**Detection:** Splunk alert on >5 failed Event ID 4625 logons across multiple accounts from single source IP within 60 seconds.

**Key Event IDs:** 4625 (failed logon), 4768 (Kerberos TGT failure)

---

### 2. GPO Privilege Escalation (T1484.001)

**Setup:** Granted a domain user write permissions over a GPO linked to the domain.

**Tool:** SharpGPOAbuse

```powershell
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount john.doe --GPOName "Workstation Policy"
```

**Detection:** Event ID 5136 (Directory Service Object Modified) on GPO change. Splunk alert on GPO modifications outside change-window hours.

**Key Event IDs:** 5136, 4732 (member added to local admin group)

---

### 3. Pass-the-Hash (T1550.002)

**Tool:** Mimikatz → CrackMapExec

```
mimikatz # sekurlsa::logonpasswords
crackmapexec smb 10.80.80.X -u administrator -H <NTLM_HASH>
```

**Detection:** Event ID 4624 Logon Type 3 with NTLMv2 authentication and no prior interactive session from the same source.

**Key Event IDs:** 4624 (Logon Type 3), 4672 (special privileges assigned)

---

### 4. Kerberoasting (T1558.003)

**Tool:** Impacket (GetUserSPNs.py) from Kali Linux

```bash
python3 GetUserSPNs.py ad.lab/john.doe:password -dc-ip 10.80.80.2 -request
```

**Detection:** Event ID 4769 with Ticket Encryption Type 0x17 (RC4) — modern environments should only see AES encryption; RC4 is a Kerberoasting indicator.

**Key Event IDs:** 4769 (Kerberos service ticket requested)

---

## Splunk Universal Forwarder

The Domain Controller forwards Windows Event Logs to Splunk (Ubuntu VM at `10.10.10.13:9997`).

**Installed on:** DC1 (10.80.80.2)
**Forwards to:** 10.10.10.13 port 9997
**Management port:** 10.10.10.13 port 8089
**Data collected:** All Windows Event Logs (Application, Security, System, Sysmon)

---

## Snapshots

Snapshots were taken for all three AD VMs at baseline configuration so the lab can be rolled back after destructive attack simulations.

| VM | Snapshot Name |
|---|---|
| Windows Server 2019 | `Baseline - AD Configured` |
| Windows 10 Enterprise VM1 | `Baseline - Domain Joined` |
| Windows 10 Enterprise VM2 | `Baseline - Domain Joined` |
