# Wazuh + Ubuntu — Complete Security Monitoring Setup Guide

> **Target OS:** Ubuntu 22.04 LTS / 24.04 LTS (also works on 20.04)  
> **Wazuh Version:** 4.14.6  
> **Date:** 2026-07-01  
> **Agent IP (example):** 192.168.1.27  
> **Manager IP (example):** 192.168.1.120 *(replace with your actual IP)*

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Find Your IP Address](#step-1--find-your-ip-address)
4. [Step 2 — Update System & Install Dependencies](#step-2--update-system--install-dependencies)
5. [Step 3 — Add Wazuh Repository](#step-3--add-wazuh-repository)
6. [Step 4 — Install Wazuh Manager (on Manager machine)](#step-4--install-wazuh-manager)
7. [Step 5 — Start and Verify Wazuh Manager](#step-5--start-and-verify-wazuh-manager)
8. [Step 6 — Install Wazuh Agent (on monitored Ubuntu machine)](#step-6--install-wazuh-agent)
9. [Step 7 — Fix the Manager IP in Agent Config](#step-7--fix-the-manager-ip-in-agent-config)
10. [Step 8 — Configure File Integrity Monitoring (FIM)](#step-8--configure-fim)
11. [Step 9 — Configure Rootcheck](#step-9--configure-rootcheck)
12. [Step 10 — Configure Syscollector](#step-10--configure-syscollector)
13. [Step 11 — Install & Configure ClamAV](#step-11--install--configure-clamav)
14. [Step 12 — Install & Configure YARA](#step-12--install--configure-yara)
15. [Step 13 — Configure Active Response](#step-13--configure-active-response)
16. [Step 14 — Install & Configure Auditd](#step-14--install--configure-auditd)
17. [Step 15 — Configure Log Collection](#step-15--configure-log-collection)
18. [Step 16 — Configure UFW Firewall Integration](#step-16--configure-ufw)
19. [Step 17 — Start the Wazuh Agent](#step-17--start-the-wazuh-agent)
20. [Step 18 — Test & Verify Everything Works](#step-18--test--verify)
21. [Troubleshooting: IP Address Fix](#troubleshooting-ip-address-fix)
22. [Ubuntu vs Kali — Key Differences](#ubuntu-vs-kali-differences)
23. [Final Configuration Summary](#final-configuration-summary)

---

## Overview

This guide documents every command needed to turn an **Ubuntu** system into a fully monitored endpoint using **Wazuh 4.14.6**. It covers:

- Wazuh Agent installation and enrollment
- File Integrity Monitoring (FIM) on `/home`, `/etc`, `/usr/bin`, `/var/www`
- Rootcheck (all modules enabled)
- Syscollector (hardware, packages, ports, processes)
- ClamAV antivirus with daily scheduled scans
- YARA malware detection with automatic quarantine
- Active Response (auto-block SSH brute force IPs)
- Auditd rules for privilege escalation & sensitive file changes
- Log collection from auth.log, syslog, auditd, ClamAV
- UFW firewall log integration

---

## Prerequisites

- Ubuntu 20.04 / 22.04 / 24.04 LTS
- Root / sudo access
- Internet connectivity (`curl`, `wget`, `gpg` available)
- A second machine running **Wazuh Manager** (or install manager on the same host)

---

## Step 1 — Find Your IP Address

Before configuring the agent, confirm your machine's IP. This is the most common setup issue.

```bash
hostname -I
```

**Expected output:**
```
192.168.1.27
```

> Use this IP as `MANAGER_IP` when configuring the Wazuh Manager. If the Manager runs on a *separate* machine, run `hostname -I` **on that machine** and use its IP.

You can also use `ip addr show` for more detail:
```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

---

## Step 2 — Update System & Install Dependencies

```bash
# Always update first on Ubuntu
sudo apt-get update && sudo apt-get upgrade -y

# Install required dependencies
sudo apt-get install -y curl gnupg apt-transport-https ca-certificates lsb-release
```

---

## Step 3 — Add Wazuh Repository

```bash
# Download the GPG key
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH -o /tmp/wazuh.key

# Import into system keyring (dearmor converts ASCII to binary GPG format)
sudo gpg --dearmor --yes -o /usr/share/keyrings/wazuh.gpg /tmp/wazuh.key

# Add the Wazuh apt repository
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

# Update package lists to pick up the new Wazuh repo
sudo apt-get update
```

Verify the repo was added:
```bash
cat /etc/apt/sources.list.d/wazuh.list
```

---

## Step 4 — Install Wazuh Manager

> Run this on the machine that will act as the **Manager** (could be the same Ubuntu machine, or a separate host).

```bash
sudo apt-get install -y wazuh-manager
```

> ⚠️ This will remove `wazuh-agent` if installed. You will reinstall the agent after the manager is set up.

---

## Step 5 — Start and Verify Wazuh Manager

```bash
# Enable and start the manager service
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-manager

# Check systemd status
sudo systemctl status wazuh-manager --no-pager

# Verify all internal processes are running
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

Verify the ports are open:
```bash
sudo ss -tlnp | grep -E "1514|1515|55000"
```

---

## Step 6 — Install Wazuh Agent

On the **monitored Ubuntu machine** (could be the same or different):

```bash
sudo apt-get install -y wazuh-agent
```

If dpkg prompts about a service file conflict, accept with `Y`:
```bash
sudo dpkg --force-confnew --configure wazuh-agent <<< 'Y'
```

---

## Step 7 — Fix the Manager IP in Agent Config

### 7a. Find your Manager's IP
```bash
hostname -I
# Example output: 192.168.1.27
```

### 7b. Open the agent configuration
```bash
sudo nano /var/ossec/etc/ossec.conf
```

### 7c. Find and update the `<client>` section
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

### 7d. Save and close
Press `Ctrl+X`, then `Y`, then `Enter`.

### Alternative — use sed to replace inline:
```bash
sudo sed -i 's/<address>MANAGER_IP<\/address>/<address>192.168.1.120<\/address>/' \
  /var/ossec/etc/ossec.conf
```

---

## Step 8 — Configure FIM

Open `/var/ossec/etc/ossec.conf` and update the `<syscheck>` block:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Replace the default directories with:

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>
  <scan_on_start>yes</scan_on_start>
  <alert_new_files>yes</alert_new_files>
  <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

  <!-- Realtime FIM with content change reporting -->
  <directories realtime="yes" report_changes="yes">/etc,/usr/bin,/usr/sbin</directories>
  <directories realtime="yes" report_changes="yes">/bin,/sbin,/boot</directories>
  <directories realtime="yes" report_changes="yes">/home</directories>
  <directories realtime="yes" report_changes="yes">/var/www</directories>
  <directories realtime="yes" report_changes="yes">/root/.ssh</directories>

  <!-- Ubuntu-specific important paths -->
  <directories realtime="yes" report_changes="yes">/etc/apt</directories>

  <!-- Files/directories to ignore -->
  <ignore>/etc/mtab</ignore>
  <ignore>/etc/hosts.deny</ignore>
  <ignore>/etc/adjtime</ignore>
  <ignore type="sregex">.log$|.swp$</ignore>

  <skip_nfs>yes</skip_nfs>
  <skip_dev>yes</skip_dev>
  <skip_proc>yes</skip_proc>
  <skip_sys>yes</skip_sys>
  <process_priority>10</process_priority>
  <max_eps>50</max_eps>
</syscheck>
```

Ensure monitored directories exist:
```bash
sudo mkdir -p /var/www
sudo mkdir -p /root/.ssh && sudo chmod 700 /root/.ssh
```

---

## Step 9 — Configure Rootcheck

Rootcheck is enabled by default. Verify in `ossec.conf`:

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
  <!-- Every 12 hours -->
  <frequency>43200</frequency>
  <rootkit_files>etc/shared/rootkit_files.txt</rootkit_files>
  <rootkit_trojans>etc/shared/rootkit_trojans.txt</rootkit_trojans>
  <skip_nfs>yes</skip_nfs>
</rootcheck>
```

---

## Step 10 — Configure Syscollector

Already enabled by default. Verify:

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

## Step 11 — Install & Configure ClamAV

### Install ClamAV
```bash
sudo apt-get install -y clamav clamav-daemon clamdscan
```

### Update virus definitions
```bash
# Stop the auto-updater temporarily to allow manual update
sudo systemctl stop clamav-freshclam

# Download latest signatures (~85MB)
sudo freshclam

# Re-start freshclam
sudo systemctl start clamav-freshclam
sudo systemctl enable clamav-freshclam
```

### Enable and start the ClamAV daemon
```bash
sudo systemctl enable --now clamav-daemon
```

Verify ClamAV daemon is running:
```bash
sudo systemctl status clamav-daemon --no-pager
```

### Create a daily scan script
```bash
sudo nano /etc/cron.daily/clamav-scan
```

Paste the following content:
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

Make it executable and create the log file:
```bash
sudo chmod +x /etc/cron.daily/clamav-scan
sudo touch /var/log/clamav/scan.log
sudo chown clamav:clamav /var/log/clamav/scan.log
```

---

## Step 12 — Install & Configure YARA

### Install YARA
```bash
# On Ubuntu, YARA is available in the default repos
sudo apt-get install -y yara
```

### Create YARA rules directory
```bash
sudo mkdir -p /var/ossec/etc/yara/rules
```

### Download a community YARA ruleset
```bash
curl -s https://raw.githubusercontent.com/Yara-Rules/rules/master/malware/APT_APT1.yar \
  -o /tmp/apt1.yar
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
        description = "Detects test malware files containing malicious_file_test_pattern"
        author = "Security Admin"
    strings:
        $test_string = "malicious_file_test_pattern"
    condition:
        $test_string
}

rule detect_webshell {
    meta:
        description = "Detects common PHP webshell patterns"
    strings:
        $a = "eval(base64_decode(" nocase
        $b = "system($_GET[" nocase
        $c = "passthru($_POST[" nocase
    condition:
        any of them
}
```

### Deploy the YARA active-response script
```bash
sudo nano /var/ossec/active-response/bin/yara.py
```

Paste this complete script:
```python
#!/usr/bin/env python3
import sys, json, subprocess, os, shutil
from datetime import datetime

LOG_FILE = "/var/ossec/logs/active-responses.log"
YARA_RULES = "/var/ossec/etc/yara/rules/local_rules.yar"
YARA_BIN = "/usr/bin/yara"
QUARANTINE_DIR = "/var/ossec/quarantine"

def log(msg):
    ts = datetime.now().strftime("%Y/%m/%d %H:%M:%S")
    with open(LOG_FILE, "a") as f:
        f.write(f"{ts} yara-active-response: {msg}\n")

def quarantine(path):
    try:
        os.makedirs(QUARANTINE_DIR, mode=0o750, exist_ok=True)
        dest = os.path.join(QUARANTINE_DIR,
               f"{os.path.basename(path)}.{datetime.now().strftime('%Y%m%d_%H%M%S')}.quarantine")
        shutil.move(path, dest)
        os.chmod(dest, 0o000)
        log(f"QUARANTINE: '{path}' → '{dest}'")
    except Exception as e:
        log(f"QUARANTINE FAILED: {e}")

def main():
    try:
        payload = json.loads(sys.stdin.read())
    except Exception as e:
        log(f"Parse error: {e}"); sys.exit(1)

    file_path = (payload.get("parameters", {})
                        .get("alert", {})
                        .get("syscheck", {})
                        .get("path"))
    if not file_path and len(sys.argv) > 3:
        file_path = sys.argv[3]
    if not file_path:
        log("No file path found in payload."); sys.exit(1)
    if not os.path.exists(file_path):
        log(f"File not found: {file_path}"); sys.exit(1)

    result = subprocess.run([YARA_BIN, YARA_RULES, file_path],
                            stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
    if result.returncode == 0 and result.stdout.strip():
        for line in result.stdout.strip().splitlines():
            log(f"YARA MATCH: {line}")
        quarantine(file_path)
    else:
        log(f"YARA clean: {file_path}")

if __name__ == "__main__":
    main()
```

Set correct permissions:
```bash
sudo chmod 750 /var/ossec/active-response/bin/yara.py
sudo chown root:wazuh /var/ossec/active-response/bin/yara.py
```

---

## Step 13 — Configure Active Response

Add these blocks to `/var/ossec/etc/ossec.conf` **on the Manager machine**:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find the command section and add:

```xml
<!-- YARA scan integration -->
<command>
  <name>yara_scan</name>
  <executable>yara.py</executable>
  <timeout_allowed>no</timeout_allowed>
</command>

<!-- Trigger YARA scan when FIM detects new/modified files -->
<active-response>
  <command>yara_scan</command>
  <location>local</location>
  <rules_id>550,554</rules_id>
</active-response>

<!-- Auto-block SSH brute force attackers for 10 minutes -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712,5720</rules_id>
  <timeout>600</timeout>
</active-response>
```

> **Rule IDs:**
> - `550` = FIM: New file created
> - `554` = FIM: File modified
> - `5712` = SSH brute force
> - `5720` = Multiple SSH authentication failures

---

## Step 14 — Install & Configure Auditd

### Install auditd
```bash
sudo apt-get install -y auditd audispd-plugins
```

### Enable auditd service
```bash
sudo systemctl enable --now auditd
```

### Configure audit rules
```bash
sudo nano /etc/audit/rules.d/wazuh-security.rules
```

Paste the following:
```
## Wazuh Security Audit Rules for Ubuntu
## Clear existing rules first
-D

## Increase buffer to handle busy systems
-b 8192

## Set failure mode to syslog
-f 1

## Monitor privilege escalation
-w /etc/sudoers -p wa -k privilege_escalation
-w /etc/sudoers.d/ -p wa -k privilege_escalation
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands

## Detect execution of suspicious binaries
-w /usr/bin/whoami -p x -k suspicious_execution
-w /usr/bin/id -p x -k suspicious_execution
-w /bin/nc -p x -k suspicious_execution
-w /usr/bin/ncat -p x -k suspicious_execution
-w /usr/bin/netcat -p x -k suspicious_execution
-w /usr/bin/wget -p x -k suspicious_execution
-w /usr/bin/curl -p x -k suspicious_execution

## Monitor changes to sensitive system files
-w /etc/passwd -p wa -k sensitive_file_change
-w /etc/shadow -p wa -k sensitive_file_change
-w /etc/group -p wa -k sensitive_file_change
-w /etc/gshadow -p wa -k sensitive_file_change
-w /etc/pam.d/ -p wa -k sensitive_file_change
-w /etc/security/ -p wa -k sensitive_file_change

## Monitor SSH configuration
-w /etc/ssh/sshd_config -p wa -k ssh_config_change
-w /home/ -p wa -k home_directory_change

## Monitor cron jobs
-w /etc/cron.d/ -p wa -k cron_change
-w /etc/crontab -p wa -k cron_change
-w /var/spool/cron/ -p wa -k cron_change

## Monitor kernel modules
-w /sbin/insmod -p x -k kernel_module
-w /sbin/modprobe -p x -k kernel_module
-w /sbin/rmmod -p x -k kernel_module

## Make rules immutable (requires reboot to change – comment out while testing)
# -e 2
```

### Load rules and verify
```bash
# Load the rules
sudo augenrules --load

# Restart auditd to apply
sudo systemctl restart auditd

# Verify rules are loaded
sudo auditctl -l

# Check auditd status
sudo systemctl status auditd --no-pager
```

---

## Step 15 — Configure Log Collection

### Ubuntu uses systemd journal + syslog

Ubuntu 22.04/24.04 by default uses **systemd-journald** and **rsyslog**. The log files to monitor:

| Log File | Contents |
|---|---|
| `/var/log/auth.log` | SSH logins, sudo, PAM |
| `/var/log/syslog` | General system messages |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/ufw.log` | UFW firewall events |
| `/var/log/dpkg.log` | Package install/remove |
| `/var/log/audit/audit.log` | Auditd events |
| `/var/log/clamav/*.log` | ClamAV scan results |

### Verify rsyslog is installed and running
```bash
sudo apt-get install -y rsyslog
sudo systemctl enable --now rsyslog
sudo systemctl status rsyslog --no-pager
```

### Add log files to Wazuh agent config

In `/var/ossec/etc/ossec.conf`, add a new `<ossec_config>` block at the bottom:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add before the final `</ossec_config>` or as a new block:

```xml
<ossec_config>
  <!-- Journald (Ubuntu systemd journal) -->
  <localfile>
    <log_format>journald</log_format>
    <location>journald</location>
  </localfile>

  <!-- SSH and authentication events -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/auth.log</location>
  </localfile>

  <!-- System messages -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/syslog</location>
  </localfile>

  <!-- Kernel messages -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/kern.log</location>
  </localfile>

  <!-- Package management -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/dpkg.log</location>
  </localfile>

  <!-- UFW Firewall events -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/ufw.log</location>
  </localfile>

  <!-- Wazuh active response actions -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/ossec/logs/active-responses.log</location>
  </localfile>

  <!-- ClamAV logs -->
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

  <!-- Auditd security audit log -->
  <localfile>
    <log_format>audit</log_format>
    <location>/var/log/audit/audit.log</location>
  </localfile>

</ossec_config>
```

---

## Step 16 — Configure UFW Firewall Integration

Ubuntu uses **UFW** (Uncomplicated Firewall) instead of iptables directly.

### Enable UFW and allow SSH
```bash
# Enable UFW
sudo ufw enable

# Allow SSH (MUST do this first or you'll lock yourself out!)
sudo ufw allow ssh

# Allow Wazuh agent communication ports
sudo ufw allow out 1514/tcp comment 'Wazuh agent communication'
sudo ufw allow out 1515/tcp comment 'Wazuh agent enrollment'

# Enable UFW logging
sudo ufw logging on

# Check status
sudo ufw status verbose
```

### Verify UFW log file exists
```bash
ls -la /var/log/ufw.log
```

---

## Step 17 — Start the Wazuh Agent

```bash
# Enable and start the agent on boot
sudo systemctl enable --now wazuh-agent

# Check status
sudo systemctl status wazuh-agent --no-pager

# Reload systemd if needed
sudo systemctl daemon-reload && sudo systemctl start wazuh-agent
```

Watch live logs:
```bash
sudo tail -f /var/ossec/logs/ossec.log
```

Check the agent is connecting:
```bash
sudo grep -E "Connected|enrollment|agentd" /var/ossec/logs/ossec.log | tail -20
```

---

## Step 18 — Test & Verify

### Test FIM (File Integrity Monitoring)
```bash
# Create → modify → delete a file in a monitored directory
echo "TEST FIM" > /home/$USER/fim_test.txt
echo "MODIFIED" >> /home/$USER/fim_test.txt
rm /home/$USER/fim_test.txt
```

### Test YARA detection
```bash
# Create a file matching the custom YARA rule
echo "malicious_file_test_pattern" > /tmp/yara_test.txt
sudo cp /tmp/yara_test.txt /home/yara_test.txt

# Watch the YARA response log
sudo tail -f /var/ossec/logs/active-responses.log
```

### Test SSH brute force detection
```bash
# From another terminal, attempt multiple failed SSH logins
for i in {1..10}; do
  ssh -o ConnectTimeout=2 -o StrictHostKeyChecking=no \
      wronguser@127.0.0.1 2>/dev/null; done
```

### Test auditd integration
```bash
# Trigger privilege escalation rules
sudo cat /etc/shadow > /dev/null

# Check the audit log
sudo ausearch -k privilege_escalation
sudo ausearch -k sensitive_file_change

# Check auditd log directly
sudo tail -f /var/log/audit/audit.log
```

### Check ClamAV is scanning
```bash
# Run the ClamAV scan script manually
sudo /etc/cron.daily/clamav-scan

# Check the results
cat /var/log/clamav/scan.log
```

### Verify all services are running
```bash
sudo systemctl status wazuh-agent --no-pager
sudo systemctl status clamav-daemon --no-pager
sudo systemctl status auditd --no-pager
sudo systemctl status rsyslog --no-pager
sudo systemctl status ufw --no-pager
```

### Check Wazuh FIM is collecting events
```bash
sudo /var/ossec/bin/agent_control -l
```

---

## Troubleshooting: IP Address Fix

### Problem
Agent logs show:
```
ERROR: (1208): Unable to connect to enrollment service at '[MANAGER_IP]:1515'
```
or
```
ERROR: (1208): Unable to connect to enrollment service at '[192.168.1.120]:1515'
```

### Solution

**Step 1 — Find your correct Manager IP:**
```bash
hostname -I
```

Output example:
```
192.168.1.27
```

**Step 2 — Edit the agent config:**
```bash
sudo nano /var/ossec/etc/ossec.conf
```

**Step 3 — Find and replace:**

Before:
```xml
<client>
  <server>
    <address>MANAGER_IP</address>
    ...
  </server>
</client>
```

After:
```xml
<client>
  <server>
    <address>192.168.1.120</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

**Or use sed (faster):**
```bash
sudo sed -i 's|<address>MANAGER_IP</address>|<address>192.168.1.120</address>|g' \
  /var/ossec/etc/ossec.conf
```

**Step 4 — Restart the agent:**
```bash
sudo systemctl restart wazuh-agent
```

**Step 5 — Verify:**
```bash
sudo tail -n 30 /var/ossec/logs/ossec.log | grep -E "Connected|enrollment|ERROR"
```

**Step 6 — Check the Manager is reachable:**
```bash
# Test port connectivity from the agent machine
nc -zv 192.168.1.120 1515
nc -zv 192.168.1.120 1514
```

**Step 7 — If Manager is on same machine, use 127.0.0.1:**
```bash
sudo sed -i 's|<address>.*</address>|<address>127.0.0.1</address>|g' \
  /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-agent
```

---

## Ubuntu vs Kali Differences

| Feature | Ubuntu | Kali Linux |
|---|---|---|
| Package manager | `apt` (same) | `apt` |
| Default firewall | **UFW** | No default firewall |
| Log: Kernel | `/var/log/kern.log` | via journald |
| Log: Auth | `/var/log/auth.log` | `/var/log/auth.log` |
| Log: Syslog | `/var/log/syslog` | `/var/log/syslog` |
| Auditd rules file | `/etc/audit/rules.d/*.rules` | Same |
| ClamAV package | `clamav clamav-daemon` | Same |
| YARA package | `yara` (official repos) | `yara` |
| Network config | networkd / netplan | NetworkManager |
| AppArmor | **Yes** (enabled by default) | Not enabled |
| `/var/log/messages` | Not created by default | Need to add to rsyslog |

### Ubuntu-specific: Check AppArmor status
```bash
sudo aa-status
```

Allow ClamAV daemon through AppArmor if blocked:
```bash
sudo aa-complain /usr/sbin/clamd
# Or fully disable for a profile:
sudo aa-disable /usr/sbin/clamd
```

### Ubuntu-specific: Enable UFW log integration
```bash
sudo ufw logging on
sudo systemctl restart ufw
```

---

## Final Configuration Summary

| Component | Status | Config File |
|---|---|---|
| Wazuh Agent 4.14.6 | ✅ Running | `/var/ossec/etc/ossec.conf` |
| Manager IP | `192.168.1.120:1514` | `<client><server>` |
| FIM (syscheck) | ✅ Realtime | `/home`, `/etc`, `/usr/bin`, `/var/www`, `/etc/apt` |
| Rootcheck | ✅ All modules | Every 12h |
| Syscollector | ✅ Full inventory | Hardware, packages, ports, processes |
| ClamAV | ✅ Daily scans | `/etc/cron.daily/clamav-scan` |
| YARA | ✅ Quarantine enabled | `/var/ossec/etc/yara/rules/` |
| Active Response | ✅ YARA + firewall-drop | Rules 550/554, 5712/5720 |
| Auditd | ✅ Extended rules | `/etc/audit/rules.d/wazuh-security.rules` |
| UFW | ✅ Enabled + logging | `/var/log/ufw.log` monitored |
| Log Collection | ✅ Full | auth.log, syslog, kern.log, ufw.log, auditd, ClamAV |
| rsyslog | ✅ Running | `/etc/rsyslog.conf` |

### Installed Packages
```
wazuh-agent 4.14.6
clamav + clamav-daemon + clamdscan
yara
auditd + audispd-plugins
rsyslog
ufw (usually pre-installed)
```

### Important File Paths
| File | Purpose |
|---|---|
| `/var/ossec/etc/ossec.conf` | Agent main configuration |
| `/var/ossec/logs/ossec.log` | Agent operational logs |
| `/var/ossec/logs/active-responses.log` | Active response log |
| `/var/ossec/active-response/bin/yara.py` | YARA active response script |
| `/var/ossec/etc/yara/rules/` | YARA rules |
| `/var/ossec/quarantine/` | Quarantined files |
| `/etc/audit/rules.d/wazuh-security.rules` | Auditd rules |
| `/etc/cron.daily/clamav-scan` | ClamAV daily scan |
| `/var/log/clamav/scan.log` | ClamAV scan results |
| `/var/log/audit/audit.log` | Auditd events |
| `/var/log/auth.log` | SSH/PAM authentication |
| `/var/log/ufw.log` | UFW firewall events |
| `/var/log/kern.log` | Kernel events |
