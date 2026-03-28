# Wazuh File Integrity Monitoring (FIM) Lab 
> **Author:** Nikitha Deshpande

---
## Overview

This lab demonstrates how to configure **Wazuh** for real-time **File Integrity Monitoring (FIM)** across Ubuntu and Windows environments, and how to integrate the **VirusTotal API** to detect potentially malicious files.

---
## Lab Environment

| Machine | Role | OS |
|--------|------|----|
| Machine A | Wazuh Server / Manager | Ubuntu 24.04 LTS |
| Machine B | Ubuntu Agent | Ubuntu 24.04 LTS |
| Machine C | Windows Agent | Windows 10 Home |

> All machines are connected to the same lab network via Oracle VirtualBox.

---
## Goals

- Configure Wazuh manager-side shared configuration to enable FIM on agents
- Demonstrate that agents receive configuration from the manager (no editing of agent `ossec.conf` required)
- Trigger and validate FIM alerts from Ubuntu and Windows agents
- Query the VirusTotal API to check for malware

---
## Part 1: File Integrity Monitoring (FIM)

### Target Directories

| Agent | Monitored Path |
|-------|----------------|
| Ubuntu (Machine B) | `/etc/tmp` |
| Windows (Machine C) | `C:\Users\Public\tmp` |

---
### Step 1 — Configure the Wazuh Manager

On **Machine A (Wazuh Server)**, edit the shared agent configuration:

```bash
sudo nano /var/ossec/etc/shared/default/agent.conf
```

Paste the following XML configuration:

```xml
<agent_config>
  <syscheck>
    <directories realtime="yes" report_changes="yes">/etc/tmp</directories>
    <directories realtime="yes" report_changes="yes">C:\Users\Public\tmp</directories>
  </syscheck>
</agent_config>
```

This pushes the monitoring configuration to all agents automatically — no need to edit `ossec.conf` on individual machines.

---

### Step 2 — Restart Wazuh Manager and Agents

**Machine A (Wazuh Server):**
```bash
sudo systemctl restart wazuh-manager
```

**Machine B (Ubuntu Agent):**
```bash
sudo systemctl restart wazuh-agent

# Verify agent logs
sudo tail -n 200 /var/ossec/logs/ossec.log
```

**Machine C (Windows Agent) — via PowerShell:**
```powershell
Restart-Service -Name "WazuhSvc"
```

Or restart via the Wazuh Agent Manager GUI.

---

### Step 3 — Create Monitored Directories and Trigger FIM Events

**On Ubuntu (Machine B):**
```bash
sudo mkdir -p /etc/tmp
sudo touch /etc/tmp/testfile.txt
echo "Hello FIM" | sudo tee /etc/tmp/testfile.txt
```

**On Windows (Machine C):**
```powershell
New-Item -ItemType Directory -Path "C:\Users\Public\tmp" -Force
New-Item -ItemType File -Path "C:\Users\Public\tmp\testfile.txt"
Set-Content -Path "C:\Users\Public\tmp\testfile.txt" -Value "Hello FIM"
```

---

### Step 4 — Verify FIM Alerts on Dashboard

Navigate to **Wazuh Dashboard → Modules → Integrity Monitoring**

Alerts should appear for file creation and modification events on both agents.

---

## Part 2: VirusTotal Integration

### Step 1 — Get a VirusTotal API Key

1. Create a free account at [https://www.virustotal.com](https://www.virustotal.com)
2. Navigate to your **Profile → API Key**
3. Copy the API key

---

### Step 2 — Add VirusTotal Integration to Wazuh

On **Machine A**, edit the Wazuh configuration file:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following block **before** the closing `</ossec_config>` tag:

```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_VIRUSTOTAL_API_KEY</api_key>
  <rule_id>554, 550</rule_id>
  <alert_format>json</alert_format>
</integration>
```

Restart the Wazuh manager:

```bash
sudo systemctl restart wazuh-manager
```

---

### Step 3 — Test with Malicious and Normal Files

Download a test malicious file (safe for testing) and place it in the monitored directory:

```bash
# On Ubuntu - place in monitored path
sudo cp <malicious_test_file> /etc/tmp/

# Create a normal file for comparison
echo "This is a normal file" | sudo tee /etc/tmp/normal.txt
```

---

### Step 4 — Verify Integration Logs

```bash
sudo cat /var/ossec/logs/integrations.log | grep -i "virustotal"
```

Entries in this log confirm active communication between Wazuh and the VirusTotal API.

---

## Results & Screenshots

### Active Agents on Wazuh Dashboard
![Active Agents](screenshots/Picture1.png)
> Both Ubuntu and Windows agents showing as **active** with 100% agent coverage.

### Integrity Monitoring Dashboard
![FIM Dashboard](screenshots/Picture2.png)
> Real-time FIM alerts showing file modification events from the Windows agent.

### Security Events Log
![Security Events](screenshots/Picture3.png)
> Security events filtered to show relevant rule IDs, excluding AppArmor noise (rule 52002).

### VirusTotal API Integration Confirmed
![VirusTotal Logs](screenshots/Picture4.png)
> Terminal output confirming active VirusTotal API communication via integration logs.

---

## Challenges Faced

### 1. AppArmor Log Flooding
When the malicious file was introduced, **rule 52002 (AppArmor)** generated a massive volume of alerts, making it extremely difficult to isolate the specific VirusTotal-related alerts (rule 87105).

**Workaround:**
- Applied a `NOT rule.id: 52002` filter in the Wazuh Dashboard
- Adjusted the time window between **Last 1 hour** and **Last 24 hours**
- Verified VirusTotal communication directly via terminal integration logs

### 2. Alert Level 12 Not Visible in Dashboard
Despite the VirusTotal integration working correctly (confirmed via logs), alert level 12 triggers were not surfacing clearly in the dashboard due to log volume.

**Workaround:**
```bash
sudo cat /var/ossec/logs/integrations.log | grep -i "virustotal"
```
This confirmed the API was responding and files were being scanned.

---

## Key Takeaways

- **Centralised configuration** via `agent.conf` makes managing multiple agents far more scalable
- **Real-time FIM** is a powerful first line of defence for detecting unauthorised file changes
- **Noise filtering** in security dashboards is a critical skill — not all alerts are equal
- **Troubleshooting at the terminal level** is often necessary when dashboards don't surface what you expect
- Security monitoring is rarely clean on the first attempt — persistence and methodical debugging matter

---

## References

- [Wazuh FIM Documentation](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
- [Wazuh VirusTotal Integration](https://documentation.wazuh.com/current/proof-of-concept-guide/detect-remove-malware-virustotal.html)
- [VirusTotal API](https://www.virustotal.com/gui/home/upload)

---

> 📌 *This lab was completed as part of the CE5022 Log Files and Event Analysis module.*
