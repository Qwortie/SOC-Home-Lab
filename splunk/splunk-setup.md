# Splunk SIEM Setup & Configuration

## Environment

| Setting | Value |
|---|---|
| Host OS | Ubuntu 22.04 LTS |
| Segment | SECURITY (10.10.10.0/24) |
| Splunk IP | 10.10.10.13 |
| Splunk Web UI | http://10.10.10.13:8000 |
| Receive Port | 9997 (Universal Forwarder data) |
| Management Port | 8089 |
| Splunk Version | 9.x Enterprise |

---

## Installation

```bash
# Install curl dependency
sudo apt install curl

# Install Splunk (.deb)
sudo dpkg -i splunk-9.x.x-linux-amd64.deb

# Start Splunk and accept license
sudo /opt/splunk/bin/splunk start --accept-license --answer-yes

# Enable auto-start on boot
sudo /opt/splunk/bin/splunk enable boot-start
```

Splunk runs at `http://127.0.0.1:8000` locally on the Ubuntu VM.

---

## Receiving Configuration

Configured Splunk to listen for incoming Universal Forwarder data:

```
Settings → Forwarding and Receiving → Receiving → Add New → Port: 9997
```

---

## Data Sources

### Windows Domain Controller (10.80.80.2)
Splunk Universal Forwarder v9.x installed on Windows Server 2019 DC.

**Forwarder config:**
- Management server: `10.10.10.13:8089`
- Receiving indexer: `10.10.10.13:9997`
- Data collected: All Windows Event Logs (Security, System, Application)

**Index name:** `windows`

**Key Event IDs ingested:**

| Event ID | Description |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4648 | Logon with explicit credentials |
| 4672 | Special privileges assigned |
| 4688 | Process creation |
| 4698 | Scheduled task created |
| 4720 | User account created |
| 4732 | Member added to security-enabled local group |
| 4769 | Kerberos service ticket requested |
| 5136 | Directory service object modified |

### Sysmon (planned)
Sysmon deployed via GPO to all AD domain machines for enriched process, network, and file telemetry.

---

## Querying Data

```spl
# View all ingested Windows events
index="windows"

# Failed logons only
index="windows" EventCode=4625

# Privileged logons
index="windows" EventCode=4672

# Process creation
index="windows" EventCode=4688
```

---

## Detection Rules

Custom correlation searches are maintained in the [splunk-detection-rules](https://github.com/Qwortie/splunk-detection-rules) repository. Key detections running in this environment:

| Rule | MITRE ID | Severity |
|---|---|---|
| Password Spray — Multiple Account Failures | T1110.003 | High |
| Pass-the-Hash — NTLM Lateral Movement | T1550.002 | High |
| Kerberoasting — RC4 Service Ticket | T1558.003 | High |
| Scheduled Task Persistence | T1053.005 | Medium |
| GPO Modification Outside Change Window | T1484.001 | Medium |
| New Local Admin Account Created | T1136.001 | Medium |
| PowerShell Encoded Command Execution | T1059.001 | High |

---

## Troubleshooting

**No data showing in Splunk after forwarder install:**
1. Wait 2–3 minutes for the first batch of events to arrive
2. Perform activity on the DC (open applications, change settings) to generate events
3. Verify the forwarder service is running on the DC: `services.msc → SplunkForwarder`
4. Confirm the SECURITY → AD_LAB firewall path is open (pfSense allows SECURITY outbound to AD_LAB)

**Splunk not starting after Ubuntu reboot:**
```bash
sudo /opt/splunk/bin/splunk start
```
Or if boot-start was enabled it should auto-start — check with:
```bash
sudo systemctl status SplunkForwarder
```
