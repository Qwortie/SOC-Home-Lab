# Incident Response Report

**Report ID:** IR-<!-- number -->
**Classification:** <!-- Confidential / Internal / Public -->
**Date of Detection:** <!-- YYYY-MM-DD -->
**Date of Report:** <!-- YYYY-MM-DD -->
**Lead Analyst:** Christopher Rice
**Status:** <!-- Open / Contained / Resolved / Closed -->
**Severity:** <!-- Critical / High / Medium / Low -->

---

## 1. Executive Summary

<!-- 1 paragraph, non-technical. Written for management.
     Answer: What happened? How bad was it? Is it contained?
     What was the business impact? What was done about it?
     Example:
     "On [date], a workstation in the AD_LAB environment was compromised
     through a phishing-style initial access vector. The attacker established
     persistence, escalated privileges to Domain Admin, and moved laterally
     to two additional hosts before detection. The incident was contained
     within [X] hours. No data exfiltration was confirmed. Affected systems
     have been isolated and remediation is underway." -->

---

## 2. Incident Overview

| Field | Detail |
|---|---|
| Incident ID | IR-<!-- number --> |
| Detection Source | <!-- Splunk Alert / Manual Discovery / EDR / etc. --> |
| Detection Time | <!-- YYYY-MM-DD HH:MM:SS --> |
| Containment Time | <!-- YYYY-MM-DD HH:MM:SS --> |
| Total Dwell Time | <!-- Time between initial compromise and detection --> |
| Initial Attack Vector | <!-- Phishing / Exploitation / Brute Force / etc. --> |
| Affected Hosts | <!-- List hostnames/IPs --> |
| Affected Accounts | <!-- List usernames --> |
| Data Impact | <!-- Confirmed exfiltration / No evidence of exfiltration --> |
| Business Impact | <!-- Describe operational impact if any --> |

---

## 3. Affected Systems

| Hostname | IP Address | OS | Role | Segment | Compromise Status |
|---|---|---|---|---|---|
| <!-- hostname --> | <!-- IP --> | <!-- OS --> | <!-- Role --> | <!-- Segment --> | <!-- Compromised / Suspected / Clean --> |

---

## 4. Attack Timeline

<!-- This is the most critical section. Document every confirmed event
     in chronological order. Be as precise as timestamps allow. -->

| Timestamp | Host | Event | Source | Event ID / Log |
|---|---|---|---|---|
| <!-- YYYY-MM-DD HH:MM:SS --> | <!-- host --> | <!-- what happened --> | <!-- Splunk / Sysmon / pfSense / etc. --> | <!-- Event ID or log reference --> |
| | | | | |
| | | | | |

---

## 5. Attack Chain Analysis

### 5.1 Initial Access
<!-- How did the attacker first get onto the system?
     Document the exact method, vulnerability, or technique used. -->

**Technique:** <!-- e.g. Phishing, Exploit, Brute Force -->
**MITRE ID:** <!-- e.g. T1566.001 -->
**Evidence:**
```
<!-- Paste relevant log entries, Splunk output, or Sysmon events here -->
```

---

### 5.2 Execution
<!-- What did the attacker run once they had access?
     Commands, scripts, binaries, PowerShell, etc. -->

**Technique:** <!-- e.g. PowerShell, WMI, Scripting -->
**MITRE ID:** <!-- e.g. T1059.001 -->
**Evidence:**
```
<!-- Paste relevant log entries here -->
```

---

### 5.3 Persistence
<!-- How did the attacker ensure they could maintain access
     after a reboot or session termination? -->

**Technique:** <!-- e.g. Registry Run Key, Scheduled Task, New User Account -->
**MITRE ID:** <!-- e.g. T1547.001 -->
**Evidence:**
```
<!-- Paste relevant log entries here -->
```

---

### 5.4 Privilege Escalation
<!-- How did the attacker elevate from their initial access level
     to a higher privilege level (e.g. local admin, domain admin)? -->

**Technique:** <!-- e.g. Token Impersonation, Pass-the-Hash, Exploit -->
**MITRE ID:** <!-- e.g. T1550.002 -->
**Evidence:**
```
<!-- Paste relevant log entries here -->
```

---

### 5.5 Defense Evasion
<!-- What did the attacker do to avoid detection?
     Log clearing, AV disabling, obfuscation, renamed binaries, etc. -->

**Technique:** <!-- e.g. Clear Event Logs, Encoded Commands, Masquerading -->
**MITRE ID:** <!-- e.g. T1070.001 -->
**Evidence:**
```
<!-- Paste relevant log entries here -->
```

---

### 5.6 Credential Access
<!-- Did the attacker harvest credentials from the system?
     Mimikatz, LSASS dump, SAM dump, browser credentials, etc. -->

**Technique:** <!-- e.g. LSASS Memory Dump, Kerberoasting -->
**MITRE ID:** <!-- e.g. T1003.001 -->
**Evidence:**
```
<!-- Paste relevant log entries here -->
```

---

### 5.7 Discovery
<!-- What did the attacker enumerate on the network or system?
     Port scans, AD queries, user enumeration, share enumeration, etc. -->

**Technique:** <!-- e.g. Network Service Scanning, AD Enumeration -->
**MITRE ID:** <!-- e.g. T1046 -->
**Evidence:**
```
<!-- Paste relevant log entries here -->
```

---

### 5.8 Lateral Movement
<!-- Did the attacker move to other systems using harvested credentials
     or exploitation techniques? -->

**Technique:** <!-- e.g. Pass-the-Hash, RDP, WMI, PsExec -->
**MITRE ID:** <!-- e.g. T1550.002 -->
**Source Host:** <!-- hostname / IP -->
**Destination Host:** <!-- hostname / IP -->
**Evidence:**
```
<!-- Paste relevant log entries here -->
```

---

### 5.9 Command & Control (if applicable)
<!-- Did the attacker establish a C2 channel?
     Beacon traffic, reverse shells, RAT communications. -->

**Technique:** <!-- e.g. Web Protocols, DNS Tunneling -->
**MITRE ID:** <!-- e.g. T1071.001 -->
**C2 Indicator:** <!-- IP, domain, or URL observed -->
**Evidence:**
```
<!-- Paste relevant log entries / network captures here -->
```

---

### 5.10 Exfiltration (if applicable)
<!-- Was any data transferred out of the environment?
     Volumes, destination, method. -->

**Technique:** <!-- e.g. Exfiltration over C2, SMTP, Cloud Storage -->
**MITRE ID:** <!-- e.g. T1041 -->
**Evidence:**
```
<!-- Paste relevant log entries here -->
```

---

## 6. MITRE ATT&CK Summary

| Phase | Tactic | Technique | ID | Detected? |
|---|---|---|---|---|
| 1 | Initial Access | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 2 | Execution | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 3 | Persistence | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 4 | Privilege Escalation | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 5 | Defense Evasion | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 6 | Credential Access | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 7 | Discovery | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 8 | Lateral Movement | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 9 | Command & Control | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |
| 10 | Exfiltration | <!-- technique --> | <!-- T-ID --> | <!-- ✅ Yes / ❌ No --> |

---

## 7. Indicators of Compromise (IOCs)

### Host-Based IOCs

| Type | Value | Host | Description |
|---|---|---|---|
| File Hash (MD5) | <!-- hash --> | <!-- host --> | <!-- description --> |
| File Path | <!-- path --> | <!-- host --> | <!-- description --> |
| Registry Key | <!-- key --> | <!-- host --> | <!-- description --> |
| Process Name | <!-- name --> | <!-- host --> | <!-- description --> |
| Scheduled Task | <!-- name --> | <!-- host --> | <!-- description --> |
| New User Account | <!-- username --> | <!-- host --> | <!-- description --> |

### Network-Based IOCs

| Type | Value | Description |
|---|---|---|
| IP Address | <!-- IP --> | <!-- e.g. C2 server --> |
| Domain | <!-- domain --> | <!-- description --> |
| URL | <!-- URL --> | <!-- description --> |
| Port | <!-- port --> | <!-- description --> |

---

## 8. Containment Actions

<!-- Document every containment step taken during the incident,
     in the order it was performed. -->

| Time | Action | Performed By | Result |
|---|---|---|---|
| <!-- HH:MM --> | <!-- e.g. Isolated Win10-User1 from AD_LAB segment at pfSense --> | Christopher Rice | <!-- Confirmed / Pending --> |
| <!-- HH:MM --> | <!-- e.g. Disabled compromised account john.doe in AD --> | Christopher Rice | <!-- Confirmed / Pending --> |
| <!-- HH:MM --> | <!-- e.g. Revoked Kerberos tickets on DC (klist purge) --> | Christopher Rice | <!-- Confirmed / Pending --> |

---

## 9. Eradication

<!-- What was done to remove the attacker's presence from the environment? -->

| Action | Host | Status |
|---|---|---|
| <!-- e.g. Removed scheduled task "WindowsUpdate" --> | <!-- host --> | <!-- ✅ Complete / ⏳ Pending --> |
| <!-- e.g. Deleted dropped binary at %APPDATA%\svchost32.exe --> | <!-- host --> | <!-- ✅ Complete / ⏳ Pending --> |
| <!-- e.g. Removed attacker-created local admin account --> | <!-- host --> | <!-- ✅ Complete / ⏳ Pending --> |
| <!-- e.g. Reset passwords for all compromised accounts --> | <!-- host --> | <!-- ✅ Complete / ⏳ Pending --> |
| <!-- e.g. Restored VM from clean snapshot --> | <!-- host --> | <!-- ✅ Complete / ⏳ Pending --> |

---

## 10. Recovery

<!-- How was the affected system returned to normal operation? -->

| Action | Status |
|---|---|
| <!-- e.g. Restored Win10-User1 from baseline snapshot --> | <!-- ✅ Complete / ⏳ Pending --> |
| <!-- e.g. Re-joined restored VM to ad.lab domain --> | <!-- ✅ Complete / ⏳ Pending --> |
| <!-- e.g. Verified Splunk forwarder resumed log shipping --> | <!-- ✅ Complete / ⏳ Pending --> |
| <!-- e.g. Confirmed no residual IOCs via post-recovery scan --> | <!-- ✅ Complete / ⏳ Pending --> |

---

## 11. Detection Gap Analysis

<!-- This section shows SOC maturity. For each phase of the attack chain,
     was it detected? If not, why not? What rule or data source would have caught it? -->

| Attack Phase | Detected? | Detection Method | Gap / Improvement |
|---|---|---|---|
| Initial Access | <!-- ✅ / ❌ --> | <!-- Splunk rule / manual / etc. --> | <!-- What was missing or could be improved --> |
| Execution | <!-- ✅ / ❌ --> | <!-- --> | <!-- --> |
| Persistence | <!-- ✅ / ❌ --> | <!-- --> | <!-- --> |
| Privilege Escalation | <!-- ✅ / ❌ --> | <!-- --> | <!-- --> |
| Defense Evasion | <!-- ✅ / ❌ --> | <!-- --> | <!-- --> |
| Lateral Movement | <!-- ✅ / ❌ --> | <!-- --> | <!-- --> |
| Exfiltration | <!-- ✅ / ❌ --> | <!-- --> | <!-- --> |

---

## 12. Recommendations

<!-- Concrete, actionable recommendations to prevent recurrence.
     Tie each one back to a phase of the attack chain. -->

| Priority | Recommendation | Addresses |
|---|---|---|
| High | <!-- e.g. Deploy Sysmon on all AD endpoints for process-level telemetry --> | Execution / Defense Evasion |
| High | <!-- e.g. Enable PowerShell Script Block Logging (Event ID 4104) --> | Execution |
| Medium | <!-- e.g. Implement LAPS for local admin password management --> | Lateral Movement |
| Medium | <!-- e.g. Enforce account lockout policy (5 attempts / 30 min) --> | Credential Access |
| Low | <!-- e.g. Audit scheduled tasks on all workstations quarterly --> | Persistence |

---

## 13. Lessons Learned

<!-- Written after the incident is fully closed.
     Honest reflection — what went well, what didn't, what you'd do differently. -->

**What went well:**
-

**What could be improved:**
-

**What would have reduced dwell time:**
-

---

## 14. Appendix

### A — Supporting Evidence
<!-- Links to PCAPs, Splunk saved searches, screenshots, memory dumps, etc. -->

| Item | Description | Location |
|---|---|---|
| <!-- e.g. Splunk search --> | <!-- description --> | <!-- path or URL --> |
| <!-- e.g. Wireshark PCAP --> | <!-- description --> | <!-- path or URL --> |

### B — References

- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [NIST SP 800-61 — Computer Security Incident Handling Guide](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
- [Splunk Detection Rules](https://github.com/Qwortie/splunk-detection-rules)
