# 🎯 Attack Detections

This directory contains documented attack simulations run against my 
[SOC Home Lab](https://github.com/Qwortie/SOC-Home-Lab) with full 
detection evidence captured from Wazuh XDR and Splunk SIEM. Each 
entry documents the attack execution, detection telemetry, and 
analyst findings — written from a defender's perspective.

All attack techniques are mapped to the **MITRE ATT&CK** framework.

---

## Lab Environment

| Component | Detail |
| --- | --- |
| Attacker | Kali Linux — 10.0.0.2 (LAN segment) |
| AD Domain | ad.lab — Windows Server 2019 DC at 10.80.80.2 |
| AD Clients | Windows 10 Enterprise x2 (10.80.80.11, 10.80.80.12) |
| EDR/XDR | Wazuh 4.14.5 — Ubuntu 22.04 at 10.10.10.3 |
| SIEM | Splunk Enterprise — Ubuntu 22.04 at 10.10.10.13 |
| Endpoint | Sysmon v15.20 — SwiftOnSecurity config |
| Firewall | pfSense CE — 6-segment network |

---

## Detection Index

| ID | Title | MITRE Technique | Tool | Platform | Status |
| --- | --- | --- | --- | --- | --- |
| [ATK-001](https://github.com/Qwortie/SOC-Home-Lab/blob/main/docs/attack-detections/password-spray-kerbrute/README.md) | Kerberos Password Spray | T1110.003 | Kerbrute | Wazuh | ✅ |

---

## Detection Format

Each detection entry contains:

| Section | Description |
| --- | --- |
| Attack Summary | Tool, technique, target accounts, and scope |
| Environment | Attacker and defender infrastructure |
| Detection | Platform, event IDs, alert count, key fields |
| Screenshots | Attack output, Wazuh alerts, expanded event detail |
| Kill Chain Position | MITRE ATT&CK tactic and technique mapping |
| Detection Queries | Wazuh DQL and Splunk SPL queries |
| Prerequisites | Audit policy and config required for detection |
| Key Findings | Analyst observations and detection gaps identified |
| Remediation | Concrete hardening recommendations |

---

## Detection Stack

### Wazuh XDR
- Agent-based endpoint telemetry via Sysmon
- 3,000+ built-in MITRE-mapped detection rules
- File integrity monitoring and active response
- Archive logging for full event capture

### Splunk SIEM
- Windows Event Log ingestion via Universal Forwarder
- Custom SPL detection rules mapped to MITRE ATT&CK
- Correlation searches and threshold-based alerting
- Dashboard visibility across all AD endpoints

---

## Related Repositories

| Repo | Description |
| --- | --- |
| [SOC-Home-Lab](https://github.com/Qwortie/SOC-Home-Lab) | Main lab repo — architecture, setup, Splunk rules |
| [incident-reports](https://github.com/Qwortie/SOC-Home-Lab/blob/main/docs/incident-reports/README.md) | Full kill chain IR reports following NIST SP 800-61 |
| [thm-writeups](https://github.com/Qwortie/thm-writeups) | TryHackMe SOC analyst path investigations |
