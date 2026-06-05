# RDP Brute Force — Detection & Automated Case Creation

**Date:** June 5, 2026
**Attacker:** Kali Linux (`10.0.0.2` — LAN segment)
**Target:** Win10-User1 (`10.80.80.11` — AD_LAB segment)
**Tool:** Hydra
**Protocol:** RDP (port 3389)
**MITRE ATT&CK:** [T1110.001 — Password Guessing](https://attack.mitre.org/techniques/T1110/001/) (Parent: [T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/))

---

## Objective

Simulate a network-based credential brute force attack against a domain-joined Windows endpoint and validate the full detection-to-case pipeline:

**Kali (attacker) → Win10-User1 (target) → Wazuh (detection) → Shuffle (automation) → TheHive (case)**

---

## Attack Execution

Hydra was run from Kali Linux targeting the RDP service on Win10-User1 using the Administrator account and the `rockyou.txt` wordlist:

```bash
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt -t 4 -V rdp://10.80.80.11
```

*Hydra executing the brute force attack against Win10-User1 RDP:*

![Hydra attack execution](https://github.com/Qwortie/SOC-Home-Lab/blob/main/docs/attack-detections/screenshots/hydra.png?raw=true)

The attack generated rapid sequential Windows Security Event **4625** (An account failed to log on) entries on Win10-User1, each recording:

- **Source Network Address:** `10.0.0.2` (Kali)
- **Workstation Name:** `kali`
- **Target Account:** `Administrator`
- **Logon Type:** 3 (Network)
- **Authentication Package:** NTLM
- **Failure Reason:** Unknown user name or bad password

---

## Wazuh Detection

The Wazuh agent on Win10-User1 forwarded the 4625 events to the Wazuh manager via the EventChannel. Two rules fired in sequence:

### Rule 60122 — Logon Failure (Level 5)
Fired on each individual failed logon attempt.

```
Rule:       60122
Level:      5
Description: Logon Failure - Unknown user or bad password
Groups:     windows, windows_security, authentication_failed
MITRE:      T1110.001 — Password Guessing (Wazuh default maps this to T1531 — incorrect)
Agent:      Win10-User1 (10.80.80.11)
Source IP:  10.0.0.2
```

### Rule 60204 — Multiple Windows Logon Failures (Level 10)
Fired after the frequency threshold was reached — confirming a brute force pattern.

```
Rule:       60204
Level:      10
Description: Multiple Windows Logon Failures
Groups:     windows, windows_security, authentication_failures
MITRE:      T1110 — Brute Force / Credential Access
Frequency:  8 failures within threshold window
Agent:      Win10-User1 (10.80.80.11)
Source IP:  10.0.0.2
```

Compliance frameworks triggered: PCI DSS 10.2.4, 10.2.5, 11.4 — GDPR IV_35.7.d — HIPAA 164.312.b — NIST 800-53 AU.14, AC.7, SI.4

---

## Shuffle Automation

The Wazuh integratord forwarded the alert to Shuffle via webhook. The three-node workflow executed automatically:

### Node 1 — Webhook Trigger
Received the Wazuh JSON payload containing the full Windows event data including the source IP.

Key fields extracted:
```
title:      "Multiple Windows Logon Failures"
rule_id:    60204
agent:      Win10-User1
ipAddress:  10.0.0.2
targetUser: Administrator
workstation: kali
```

### Node 2 — VirusTotal Enrichment
Queried the VirusTotal v3 IP reputation API against the source IP `10.0.0.2`:

```
GET https://www.virustotal.com/api/v3/ip_addresses/10.0.0.2
Status: 200 OK
```

Result — `10.0.0.2` is a private RFC1918 address (Kali Linux management VM):
```json
{
  "malicious": 0,
  "suspicious": 0,
  "undetected": 36,
  "harmless": 55,
  "timeout": 0
}
```

A clean VT result for the source IP indicates **internal origin** — consistent with lateral movement or an insider threat scenario rather than an external attacker. This context is embedded directly in the TheHive case.

### Node 3 — TheHive Alert Creation
Shuffle POSTed an enriched alert to TheHive's API automatically.

*Shuffle debug showing both nodes completing successfully — VT enrichment and TheHive case creation:*

![Shuffle pipeline execution](https://github.com/Qwortie/SOC-Home-Lab/blob/main/docs/attack-detections/screenshots/shuffle-debug.png?raw=true)

---

## TheHive Case

Alert created in the SOC organisation within seconds of the Wazuh detection:

```
Title:       Logon Failure - Unknown user or bad password
Reference:   1780673742.12141839
Source:      wazuh
Tags:        wazuh | brute-force | 10.0.0.2
Severity:    MEDIUM (SEV:MEDIUM)
TLP:         AMBER
PAP:         AMBER
Status:      New
Created by:  Shuffle
Occurred:    05/06/2026 12:34

Description:
  WAZUH Alert - Rule: 60122
  Agent: Win10-User1
  Source IP: 10.0.0.2
  Target User: Administrator
  Workstation: kali
  VirusTotal: {"malicious": 0, "suspicious": 0, "undetected": 36, "harmless": 55, "timeout": 0}
```

The alert was created with zero manual intervention — the analyst opens TheHive and finds a pre-populated case with the attacker IP, targeted account, source workstation, and VirusTotal reputation context already embedded.

*TheHive alert created automatically by Shuffle with full context embedded:*

![TheHive alert](https://raw.githubusercontent.com/Qwortie/SOC-Home-Lab/refs/heads/main/docs/attack-detections/screenshots/hive-alert.png)

---

## Analyst Triage Notes

| Field | Value | Assessment |
|-------|-------|------------|
| Source IP | 10.0.0.2 | Internal — Kali Linux (LAN segment) |
| VT Verdict | 0 malicious | Internal RFC1918 — expected clean result |
| Target Account | Administrator | High-value target — built-in admin account |
| Logon Type | 3 (Network) | Remote authentication attempt |
| Auth Package | NTLM | Legacy authentication — no Kerberos |
| Workstation | kali | Confirms attack origin |
| Rule Frequency | 8+ failures | Confirms automated brute force pattern |

**Verdict:** Confirmed brute force attack originating from the LAN management segment against a domain endpoint. Internal source IP with clean VT result is consistent with a compromised internal host or authorized red team activity. Escalate for investigation of the Kali host and review of lateral movement paths from LAN to AD_LAB.

> **Note on MITRE mapping:** Wazuh's default ruleset maps rule 60122 to T1531 (Account Access Removal). This is incorrect — T1531 covers adversaries deleting or disabling accounts to deny access, not failed logon attempts. The accurate technique for this attack pattern is **T1110.001 — Password Guessing**, a sub-technique of T1110 (Brute Force), since Hydra was attempting multiple passwords against a single target account.

---

## Detection Coverage

| Stage | Tool | Evidence |
|-------|------|---------|
| Attack execution | Hydra on Kali | `hydra -l Administrator rdp://10.80.80.11` |
| Event logged | Windows Security Log | Event ID 4625 — failed logon |
| Agent forwarding | Wazuh Agent (Win10-User1) | EventChannel → Wazuh manager |
| Rule fired | Wazuh Manager | Rules 60122 (level 5) → 60204 (level 10) |
| Webhook fired | Wazuh integratord | `/var/ossec/logs/integrations.log` |
| Workflow executed | Shuffle | Webhook → VT → TheHive |
| IOC enriched | VirusTotal API | IP reputation lookup |
| Case created | TheHive | Alert in SOC org — Status: New |

---

## Firewall Rule Note

The attack traversed pfSense from the LAN segment (`10.0.0.0/24`) to AD_LAB (`10.80.80.0/24`). The LAN ruleset permits full outbound access from Kali to all segments — this is intentional for the management/attack VM. In a production environment, management hosts would be restricted to specific administrative ports only.

---

## Related

- [SOAR Pipeline README](../soar-pipeline/README.md) — Full pipeline architecture and configuration
- [Password Spray Detection](./attack-detections/password-spray-kerbrute/README.md) — SMB credential attack detection
