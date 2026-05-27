# Detection: Kerberos Password Spray via Kerbrute

## Attack Summary
Simulated a password spray attack against Active Directory using 
Kerbrute from Kali Linux targeting domain user accounts.

## Environment
- Attacker:  Kali Linux (10.0.0.2) — LAN segment
- Target:    DOMAINCONTROL1 (10.80.80.2) — AD_LAB segment
- Detection: Wazuh 4.14.5 — SECURITY segment (10.10.10.3)

## Attack Details
**Tool:** Kerbrute v1.0.3  
**Technique:** MITRE ATT&CK T1110.003 — Password Spraying  
**Target accounts:** administrator, john.doe, jane.doe, guest  
**Passwords tried:** Password1, Password123, Welcome1, Summer2024  

## Detection
**Platform:** Wazuh XDR  
**Event ID:** 4771 — Kerberos Pre-Authentication Failed  
**Alert count:** 9 events within 15-minute window  

### Key Fields Captured
| Field | Value |
|---|---|
| targetUserName | john.doe |
| serviceName | krbtgt/AD.LAB |
| sourceIP | 10.0.0.2 (Kali) |
| preAuthType | 2 |
| eventID | 4771 |
| computer | DOMAINCONTROL1.ad.lab |

## Screenshots
### Wazuh 4771 Detection — 28 Hits
<img width="945" height="611" alt="eventcount" src="https://github.com/user-attachments/assets/9139641e-ce8c-40f1-805e-6b71e7a3fd7f" />

### Expanded Event Detail
<img width="940" height="722" alt="eventdetails" src="https://github.com/user-attachments/assets/f0a7501f-04d3-4222-a332-c2f85ebadffd" />

### Kerbrute Attack Output
<img width="723" height="1010" alt="kerbrute" src="https://github.com/user-attachments/assets/228ac9bc-c9bb-436e-839f-8aff8e3a98f4" />

### Wazuh Agents Active
<img width="1186" height="428" alt="agentsummary" src="https://github.com/user-attachments/assets/a79cc24b-4116-4657-85ac-a9db5651e383" />

## Kill Chain Position
Initial Access → Credential Access
T1110.003 — Brute Force: Password Spraying

## Detection Query (Wazuh DQL)
agent.name: DC1 AND data.win.system.eventID: 4771

## Detection Query (Splunk SPL)
index=windows EventCode=4771
| stats count by targetUserName, IpAddress
| where count > 3

## Prerequisites for Detection
- Windows audit policy: Kerberos Authentication Service —
  Failure auditing enabled via auditpol or GPO
- Wazuh agent deployed on DC with Security event log collection
- Sysmon deployed with SwiftOnSecurity config
- Archive logging enabled (logall_json: yes in ossec.conf)

## Key Finding — Default Audit Policy Gap
Default Windows Server 2019 audit policy does NOT enable failure 
auditing for Kerberos Authentication Service. This must be 
explicitly configured:

```powershell
auditpol /set /subcategory:"Kerberos Authentication Service" /failure:enable
auditpol /set /subcategory:"Credential Validation" /failure:enable
```

Without this setting, Event ID 4771 is silently dropped and the 
spray is completely undetectable at the OS level — regardless of 
SIEM or EDR tooling deployed.

## Remediation Recommendations
- Enable account lockout policy (threshold: 5 attempts)
- Alert on 3+ Event ID 4771 events from same source IP within 
  5 minutes
- Enable Kerberos failure auditing via GPO across all DCs
- Consider Microsoft Defender for Identity for advanced Kerberos 
  attack detection (Golden Ticket, DCSync, Kerberoasting)
- Implement time-based access restrictions for service accounts

## Tools Used
| Tool | Purpose |
|---|---|
| Kerbrute v1.0.3 | Password spray execution |
| Wazuh 4.14.5 | EDR/XDR detection |
| Splunk | SIEM correlation |
| Sysmon | Endpoint telemetry |
| pfSense CE | Network segmentation |

## MITRE ATT&CK Mapping
| ID | Technique | Tactic |
|---|---|---|
| T1110.003 | Brute Force: Password Spraying | Credential Access |
| T1078 | Valid Accounts | Defense Evasion |
| T1558 | Steal or Forge Kerberos Tickets | Credential Access |
