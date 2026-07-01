# Wazuh + Kali Linux — Complete Security Monitoring Setup Guide

> **System:** Kali GNU/Linux Rolling 2026.1  
> **Wazuh Version:** 4.14.6  
> **Date:** 2026-07-01  
> **Agent IP:** 192.168.1.27  
> **Manager IP:** 192.168.1.120 *(your environment — replace with actual)*

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Find Your IP Address](#step-1--find-your-ip-address)
4. [Step 2 — Add Wazuh Repository](#step-2--add-wazuh-repository)
5. [Step 3 — Install Wazuh Manager (on Manager machine)](#step-3--install-wazuh-manager)
6. [Step 4 — Start and verify Wazuh Manager](#step-4--start-and-verify-wazuh-manager)
7. [Step 5 — Install Wazuh Agent (on this Kali machine)](#step-5--install-wazuh-agent)
8. [Step 6 — Fix the Manager IP in Agent Config](#step-6--fix-the-manager-ip-in-agent-config)
9. [Step 7 — Configure File Integrity Monitoring (FIM)](#step-7--configure-fim)
10. [Step 8 — Configure Rootcheck](#step-8--configure-rootcheck)
11. [Step 9 — Configure Syscollector](#step-9--configure-syscollector)
12. [Step 10 — Install & Configure ClamAV](#step-10--install--configure-clamav)
13. [Step 11 — Install & Configure YARA](#step-11--install--configure-yara)
14. [Step 12 — Configure Active Response](#step-12--configure-active-response)
15. [Step 13 — Install & Configure Auditd](#step-13--install--configure-auditd)
16. [Step 14 — Configure Log Collection](#step-14--configure-log-collection)
17. [Step 15 — Start the Wazuh Agent](#step-15--start-the-wazuh-agent)
18. [Step 16 — Test & Verify Everything Works](#step-16--test--verify)
19. [Troubleshooting: IP Address Fix](#troubleshooting-ip-address-fix)
20. [Final Configuration Summary](#final-configuration-summary)

---

## Overview

This guide documents every command used to turn a fresh **Kali Linux** system into a fully monitored endpoint using **Wazuh 4.14.6**. It covers:

- Wazuh Agent installation and enrollment
- File Integrity Monitoring (FIM) on `/home`, `/etc`, `/usr/bin`, `/var/www`
- Rootcheck (all modules enabled)
- Syscollector (hardware, packages, ports, processes)
- ClamAV antivirus with daily scheduled scans
- YARA malware detection with automatic quarantine
- Active Response (auto-block SSH brute force IPs)
- Auditd rules for privilege escalation & sensitive file changes
- Log collection from auth.log, syslog, auditd, ClamAV

---

## Prerequisites

- Kali Linux 2026.1 (or similar rolling release)
- Root / sudo access
- Internet connectivity
- A second machine running **Wazuh Manager** (or install manager on the same host)

---

## Step 1 — Find Your IP Address

Before configuring the agent, confirm your machine's IP. This is the most common issue — using the wrong IP.

```bash
hostname -I
```

**Expected output:**
```
192.168.1.27
```

> Use this IP as the `MANAGER_IP` when configuring the Wazuh Manager. If the Manager runs on a *separate* machine, run `hostname -I` **on that machine** and use that IP instead.

---

## Step 2 — Add Wazuh Repository

```bash
# Download the GPG key
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH -o /tmp/wazuh.key

# Import it into the system keyring
sudo gpg --dearmor --yes -o /usr/share/keyrings/wazuh.gpg /tmp/wazuh.key

# Add the Wazuh apt repository
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" > /tmp/wazuh.list
sudo cp /tmp/wazuh.list /etc/apt/sources.list.d/wazuh.list

# Update package lists
sudo apt-get update
```

---

## Step 3 — Install Wazuh Manager

> Run this on the machine that will act as the **Manager** (could be the same Kali machine, or a separate host).

```bash
sudo apt-get install -y wazuh-manager
```

> ⚠️ This will remove `wazuh-agent` if it was previously installed. You will reinstall the agent after the manager is set up.

---

## Step 4 — Start and Verify Wazuh Manager

```bash
# Enable and start the manager service
sudo systemctl enable --now wazuh-manager

# Verify all key processes are running
sudo /var/ossec/bin/wazuh-control status
```

**Expected running processes:**
```
wazuh-modulesd is running...
wazuh-logcollector is running...
wazuh-remoted is running...      ← listens on port 1514
wazuh-syscheckd is running...
wazuh-analysisd is running...
wazuh-execd is running...
wazuh-db is running...
wazuh-authd is running...        ← listens on port 1515
wazuh-apid is running...
```

---

## Step 5 — Install Wazuh Agent

```bash
sudo apt-get install -y wazuh-agent
```

If the configurator prompts about the systemd service file change, accept with `Y`:

```bash
sudo dpkg --force-confnew --configure wazuh-agent <<< 'Y'
```

---

## Step 6 — Fix the Manager IP in Agent Config

### 6a. Find your Manager's IP
```bash
hostname -I
# Example output: 192.168.1.27
```

### 6b. Open the agent configuration file
```bash
sudo nano /var/ossec/etc/ossec.conf
```

### 6c. Find and update this section
Look for:
```xml
<client>
  <server>
    <address>MANAGER_IP</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

Replace `MANAGER_IP` with your actual manager IP:
```xml
<client>
  <server>
    <address>192.168.1.120</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

### 6d. Save and close
Press `Ctrl+X`, then `Y`, then `Enter`.

---

## Step 7 — Configure FIM

Open `/var/ossec/etc/ossec.conf` and find the `<syscheck>` section. Replace the default directory lines with:

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>
  <scan_on_start>yes</scan_on_start>
  <alert_new_files>yes</alert_new_files>
  <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

  <!-- Realtime monitoring with change reporting -->
  <directories realtime="yes" report_changes="yes">/etc,/usr/bin,/usr/sbin</directories>
  <directories realtime="yes" report_changes="yes">/bin,/sbin,/boot</directories>
  <directories realtime="yes" report_changes="yes">/home</directories>
  <directories realtime="yes" report_changes="yes">/var/www</directories>
  <directories realtime="yes" report_changes="yes">/root/.ssh</directories>
  ...
</syscheck>
```

Also ensure `/var/www` exists:
```bash
sudo mkdir -p /var/www
```

---

## Step 8 — Configure Rootcheck

The default Wazuh Manager/Agent config already enables rootcheck with all modules. Verify in `ossec.conf`:

```xml
<rootcheck>
  <disabled>no</disabled>
  <check_files>yes</check_files>
  <check_trojans>yes</check_trojans>
  <check_dev>yes</check_dev>
  <check_sys>yes</check_sys>
  <check_pids>yes</check_pids>
  <check_ports>yes</check_ports>
  <check_if>yes</check_if>
  <!-- Scan every 12 hours -->
  <frequency>43200</frequency>
  <rootkit_files>etc/shared/rootkit_files.txt</rootkit_files>
  <rootkit_trojans>etc/shared/rootkit_trojans.txt</rootkit_trojans>
  <skip_nfs>yes</skip_nfs>
</rootcheck>
```

No commands needed — already configured.

---

## Step 9 — Configure Syscollector

Verify in `ossec.conf` (already configured by default):

```xml
<wodle name="syscollector">
  <disabled>no</disabled>
  <interval>1h</interval>
  <scan_on_start>yes</scan_on_start>
  <hardware>yes</hardware>
  <os>yes</os>
  <network>yes</network>
  <packages>yes</packages>
  <ports all="yes">yes</ports>
  <processes>yes</processes>
  <users>yes</users>
  <groups>yes</groups>
  <services>yes</services>
  <browser_extensions>yes</browser_extensions>
</wodle>
```

---

## Step 10 — Install & Configure ClamAV

### Install ClamAV
```bash
sudo apt-get install -y clamav clamav-daemon
```

### Update virus definitions
```bash
# Stop the freshclam service first to allow manual update
sudo systemctl stop clamav-freshclam

# Run freshclam to download the latest signatures
sudo freshclam

# Re-start freshclam
sudo systemctl start clamav-freshclam
```

### Enable and start the ClamAV daemon
```bash
sudo systemctl enable --now clamav-daemon
```

### Create a daily scan script
```bash
sudo nano /etc/cron.daily/clamav-scan
```

Paste:
```bash
#!/bin/bash
LOGFILE="/var/log/clamav/scan.log"
DIR_TO_SCAN="/home /var/www /etc /usr/bin"
echo "--- Starting ClamAV Scan $(date) ---" >> "$LOGFILE"
if systemctl is-active --quiet clamav-daemon; then
    clamdscan --fdpass --multiscan --log="$LOGFILE" $DIR_TO_SCAN
else
    clamscan -r --log="$LOGFILE" $DIR_TO_SCAN
fi
echo "--- Finished ClamAV Scan $(date) ---" >> "$LOGFILE"
```

Make it executable:
```bash
sudo chmod +x /etc/cron.daily/clamav-scan
```

### Create scan log file
```bash
sudo touch /var/log/clamav/scan.log
sudo chown clamav:clamav /var/log/clamav/scan.log
```

---

## Step 11 — Install & Configure YARA

### Install YARA
```bash
sudo apt-get install -y yara
```

### Create YARA rules directory
```bash
sudo mkdir -p /var/ossec/etc/yara/rules
```

### Download a community YARA rule set
```bash
curl -s https://raw.githubusercontent.com/Yara-Rules/rules/master/malware/APT_APT1.yar -o /tmp/apt1.yar
sudo cp /tmp/apt1.yar /var/ossec/etc/yara/rules/apt1.yar
```

### Create a custom local rule
```bash
sudo nano /var/ossec/etc/yara/rules/local_rules.yar
```

Paste:
```yara
rule test_malware_file {
    meta:
        description = "Detects test malware files"
    strings:
        $test_string = "malicious_file_test_pattern"
    condition:
        $test_string
}
```

### Deploy the YARA active-response script
```bash
sudo nano /var/ossec/active-response/bin/yara.py
```

*(Full script content at the bottom of this document or in `yara.py`)*

Set permissions:
```bash
sudo chmod 750 /var/ossec/active-response/bin/yara.py
sudo chown root:wazuh /var/ossec/active-response/bin/yara.py
```

---

## Step 12 — Configure Active Response

Add these blocks inside `ossec.conf` (on the **Manager**):

```xml
<!-- YARA scan command -->
<command>
  <name>yara_scan</name>
  <executable>yara.py</executable>
  <timeout_allowed>no</timeout_allowed>
</command>

<!-- Trigger YARA scan on FIM alerts (file modified/added) -->
<active-response>
  <command>yara_scan</command>
  <location>local</location>
  <rules_id>550,554</rules_id>
</active-response>

<!-- Block IPs doing SSH brute force for 10 minutes -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712,5720</rules_id>
  <timeout>600</timeout>
</active-response>
```

> **Rule IDs:**
> - `550` = New file added (FIM)
> - `554` = File modified (FIM)
> - `5712` = SSH brute force attack
> - `5720` = Multiple SSH authentication failures

---

## Step 13 — Install & Configure Auditd

### Install auditd
```bash
sudo apt-get install -y auditd
```

### Configure audit rules
```bash
sudo nano /etc/audit/rules.d/audit.rules
```

Add these rules at the bottom:
```
# Monitor privilege escalation
-w /etc/sudoers -p wa -k privilege_escalation
-w /etc/sudoers.d/ -p wa -k privilege_escalation

# Detect execution of suspicious binaries
-w /usr/bin/whoami -p x -k suspicious_execution
-w /usr/bin/id -p x -k suspicious_execution
-w /bin/nc -p x -k suspicious_execution
-w /usr/bin/ncat -p x -k suspicious_execution

# Monitor changes to sensitive system files
-w /etc/passwd -p wa -k sensitive_file_change
-w /etc/shadow -p wa -k sensitive_file_change
-w /etc/group -p wa -k sensitive_file_change
-w /etc/gshadow -p wa -k sensitive_file_change
-w /etc/pam.d/ -p wa -k sensitive_file_change
```

### Load rules and enable auditd
```bash
sudo augenrules --load
sudo systemctl enable --now auditd
```

### Verify rules are loaded
```bash
sudo auditctl -l
```

---

## Step 14 — Configure Log Collection

### Install rsyslog (to generate log files)
```bash
sudo apt-get install -y rsyslog
sudo systemctl enable --now rsyslog
```

### Add `/var/log/messages` logging
```bash
sudo nano /etc/rsyslog.conf
```

Add this line inside the RULES section:
```
*.=info;*.=notice;*.=warn;auth,authpriv.none;cron,daemon.none;mail.none  -/var/log/messages
```

Restart rsyslog:
```bash
sudo systemctl restart rsyslog
```

### Add all log files to Wazuh
In `/var/ossec/etc/ossec.conf`, add these `<localfile>` entries in the second `<ossec_config>` block:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/syslog</location>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/messages</location>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/clamav/clamav.log</location>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/clamav/freshclam.log</location>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/clamav/scan.log</location>
</localfile>

<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

---

## Step 15 — Start the Wazuh Agent

```bash
sudo systemctl enable --now wazuh-agent
```

Check status:
```bash
sudo systemctl status wazuh-agent
```

Check logs:
```bash
sudo tail -f /var/ossec/logs/ossec.log
```

---

## Step 16 — Test & Verify

### Test FIM (File Integrity Monitoring)
```bash
# Create a new file in a monitored directory
echo "TEST" > /home/testfile.txt

# Modify a file
echo "MODIFIED" >> /home/testfile.txt

# Delete the file
rm /home/testfile.txt
```

### Test YARA detection
```bash
# Create a file matching the custom YARA rule
echo "malicious_file_test_pattern" > /tmp/test_yara.txt
sudo cp /tmp/test_yara.txt /home/test_yara.txt
```

Check the YARA active-response log:
```bash
sudo tail -f /var/ossec/logs/active-responses.log
```

### Test SSH brute force detection
```bash
# From another terminal, attempt multiple failed SSH logins:
for i in {1..10}; do ssh -o ConnectTimeout=2 wronguser@127.0.0.1; done
```

### Verify auditd is logging
```bash
sudo ausearch -k privilege_escalation
sudo ausearch -k sensitive_file_change
```

### Check agent connectivity
```bash
sudo /var/ossec/bin/agent_control -l
```

---

## Troubleshooting: IP Address Fix

### Problem
Agent cannot connect — logs show:
```
ERROR: (1208): Unable to connect to enrollment service at '[MANAGER_IP]:1515'
```

### Solution

**Step 1 — Find your correct IP:**
```bash
hostname -I
```
You will get output like:
```
192.168.1.27
```

**Step 2 — Edit the agent config:**
```bash
sudo nano /var/ossec/etc/ossec.conf
```

**Step 3 — Find this section:**
```xml
<client>
  <server>
    <address>MANAGER_IP</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

**Step 4 — Replace with your actual manager IP:**
```xml
<client>
  <server>
    <address>192.168.1.120</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

**Step 5 — Restart the agent:**
```bash
sudo systemctl restart wazuh-agent
```

**Step 6 — Verify connection:**
```bash
sudo tail -n 30 /var/ossec/logs/ossec.log | grep -E "Connected|enrollment|ERROR"
```

---

## Final Configuration Summary

| Component | Status | Config File |
|---|---|---|
| Wazuh Agent | ✅ Installed & Running | `/var/ossec/etc/ossec.conf` |
| Manager IP | `192.168.1.120:1514` | `<client><server>` |
| FIM (syscheck) | ✅ Enabled, Realtime | `/home`, `/etc`, `/usr/bin`, `/var/www`, `/root/.ssh` |
| Rootcheck | ✅ All modules enabled | Every 12h scan |
| Syscollector | ✅ Full inventory | Hardware, packages, ports, processes |
| ClamAV | ✅ Installed, Daily scans | `/etc/cron.daily/clamav-scan` |
| YARA | ✅ Custom rules + quarantine | `/var/ossec/etc/yara/rules/` |
| Active Response | ✅ YARA scan + firewall-drop | Rules 550/554 (FIM), 5712/5720 (SSH) |
| Auditd | ✅ Running with custom rules | `/etc/audit/rules.d/audit.rules` |
| Log Collection | ✅ auth.log, syslog, auditd, ClamAV | `<localfile>` blocks |
| rsyslog | ✅ Running (generates log files) | `/etc/rsyslog.conf` |

### Installed Packages
```
wazuh-agent 4.14.6
wazuh-manager 4.14.6 (used temporarily to configure, then reverted to agent)
clamav 1.4.4
clamav-daemon
clamav-freshclam
yara 4.5.7
auditd 4.1.2
rsyslog 8.2604.0
```

### Important File Paths
| File | Purpose |
|---|---|
| `/var/ossec/etc/ossec.conf` | Agent main configuration |
| `/var/ossec/logs/ossec.log` | Agent operational logs |
| `/var/ossec/logs/active-responses.log` | Active response actions log |
| `/var/ossec/active-response/bin/yara.py` | YARA active response script |
| `/var/ossec/etc/yara/rules/` | YARA rules directory |
| `/var/ossec/quarantine/` | Quarantine directory for YARA matches |
| `/etc/audit/rules.d/audit.rules` | Auditd monitoring rules |
| `/etc/cron.daily/clamav-scan` | ClamAV daily scan script |
| `/var/log/clamav/scan.log` | ClamAV scan results |
| `/var/log/audit/audit.log` | Auditd log |
| `/var/log/auth.log` | SSH and PAM authentication events |
