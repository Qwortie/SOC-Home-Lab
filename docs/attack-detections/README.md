# 🎯 Attack Detections

This directory contains documented attack simulations run against my 
[SOC Home Lab](https://github.com/Qwortie/SOC-Home-Lab) with full 
detection evidence captured from Wazuh XDR, Splunk SIEM, and the automated 
SOAR pipeline. Each entry documents the attack execution, detection telemetry, 
and analyst findings — written from a defender's perspective.

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
| SOAR | Shuffle + TheHive — Ubuntu 24.04 at 10.10.10.4 |
| Endpoint | Sysmon v15.20 — SwiftOnSecurity config |
| Firewall | pfSense CE — 6-segment network |

---

## Detection Index

| ID | Title | MITRE Technique | Tool | Platform | Pipeline | Status |
| --- | --- | --- | --- | --- | --- | --- |
| [ATK-001](./password-spray-kerbrute/README.md) | Kerberos Password Spray | T1110.003 | Kerbrute | Wazuh / Splunk | — | ✅ |
| [ATK-002](./rdp-brute-force-soar-pipeline.md) | RDP Brute Force | T1110.001 | Hydra | Wazuh | Wazuh → Shuffle → TheHive | ✅ |

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

### SOAR Pipeline — Wazuh → Shuffle → TheHive
- Wazuh alerts forwarded to Shuffle via webhook integration
- VirusTotal API enrichment on source IP IOCs
- Automated TheHive case creation with full alert context embedded
- Zero manual intervention from detection to case management
- See: [soar-pipeline/README.md](../../soar-pipeline/README.md)

---

## Related Repositories

| Repo | Description |
| --- | --- |
| [SOC-Home-Lab](https://github.com/Qwortie/SOC-Home-Lab) | Main lab repo — architecture, setup, Splunk rules |
| [soar-pipeline](https://github.com/Qwortie/SOC-Home-Lab/tree/main/soar-pipeline) | SOAR pipeline — Wazuh → Shuffle → TheHive with VirusTotal enrichment |
| [incident-reports](https://github.com/Qwortie/SOC-Home-Lab/blob/main/docs/incident-reports/README.md) | Full kill chain IR reports following NIST SP 800-61 |
| [thm-writeups](https://github.com/Qwortie/thm-writeups) | TryHackMe SOC analyst path investigations |
