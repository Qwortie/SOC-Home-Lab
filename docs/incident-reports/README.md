# 📑 Incident Response Reports

This directory contains structured incident response reports produced from attack simulations run in my [SOC Home Lab](https://github.com/Qwortie/soc-home-lab). Each report documents a simulated attack from detection through remediation, written from the perspective of a SOC analyst responding to a real incident.

Reports follow the **NIST SP 800-61** incident response lifecycle:
**Preparation → Detection → Containment → Eradication → Recovery → Lessons Learned**

All attack techniques are mapped to the **MITRE ATT&CK** framework.

---

## Lab Environment

| Component | Detail |
|---|---|
| Attacker | Kali Linux — 10.0.0.2 (LAN segment) |
| AD Domain | ad.lab — Windows Server 2019 DC at 10.80.80.2 |
| AD Clients | Windows 10 Enterprise x2 (10.80.80.11+) |
| SIEM | Splunk Enterprise — Ubuntu 22.04 at 10.10.10.13 |
| Firewall | pfSense CE 2.7.2 |

---

## Report Index

| ID | Title | MITRE Techniques | Severity | Status |
|---|---|---|---|---| 
| [IR-001](./IR-001-tempest-full-chain.md) | Tempest Full Attack Chain | T1566.001, T1059.001, T1547.001, T1134.001, T1027 / T1059.001, T1552.001,  T1046 / T1082,  T1572, T1071.001 | Critical | ✅  Complete |
| [IR-002](./IR-002-boogeyman.md) | Operation Boogeyman — 3-Wave Campaign | T1566, T1059, T1053, T1548, T1003, T1048, T1486 + 12 more | Critical✅ | Complete |
---

## Report Types

### Single Technique Reports (IR-00X)
Document a single attack technique — detection, response, and remediation. Used when simulating isolated TTPs to validate a specific Splunk detection rule.

### Full Attack Chain Reports (IR-00X-CHAIN)
Document a complete multi-stage attack from initial access through lateral movement. Maps the entire kill chain, records dwell time, and includes a detection gap analysis showing which phases were caught and which were missed.

---

## Report Format

Each report contains the following sections:

| Section | Description |
|---|---|
| Executive Summary | Non-technical overview of the incident and outcome |
| Incident Overview | Key metrics — detection time, dwell time, affected hosts |
| Attack Timeline | Chronological log of every confirmed event |
| Attack Chain Analysis | Per-phase breakdown mapped to MITRE ATT&CK |
| IOCs | Host-based and network-based indicators of compromise |
| Containment & Eradication | Actions taken to stop and remove the attacker |
| Detection Gap Analysis | Which phases were caught, which were missed, and why |
| Recommendations | Concrete steps to prevent recurrence |
| Lessons Learned | Honest reflection on what worked and what didn't |

---

## Detection Rules

The Splunk SPL queries used to detect the techniques documented in these reports are maintained in the [splunk-detection-rules](https://github.com/Qwortie/splunk-detection-rules) repository.
** COMING SOON **

---

## Planned Simulations

| ID | Scenario | MITRE Techniques | Status |
|---|---|---|---|
| IR-001 | Full Attack Chain | 10 Techniques | ✅ Complete |
| IR-002 | Pass-the-Hash Lateral Movement | T1550.002 | 🔄 Pending |
| IR-003 | Kerberoasting | T1558.003 | 🔄 Pending |
| IR-004 | Scheduled Task Persistence | T1053.005 | 🔄 Pending |
| IR-005 | Password Spraying | T1110.003 | 🔄 Pending |

