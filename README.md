# Linux Systems Administration & Automation Lab
 
A hands-on Linux systems administration lab built on Ubuntu (VirtualBox) simulating enterprise server operations. This lab covers user and group management, SSH hardening, sudo policy enforcement, firewall configuration, systemd service management, Bash automation, and real-world troubleshooting scenarios.
 
---
 
## Lab Environment
 
| Component | Details |
|---|---|
| Host OS | Windows 11 (dual boot) |
| Virtualization | Oracle VirtualBox |
| Guest OS | Ubuntu 24.04.2 LTS |
| VM Name | LinuxVM |
| Resources | 2 vCPUs, 2GB RAM, 25GB disk |
 
---
 
## Table of Contents
 
1. [VM Setup](#1-vm-setup)
2. [User & Group Management](#2-user--group-management)
3. [SSH Hardening](#3-ssh-hardening)
4. [Sudo Policy Configuration](#4-sudo-policy-configuration)
5. [Firewall Configuration with firewalld](#5-firewall-configuration-with-firewalld)
6. [Systemd Service Management](#6-systemd-service-management)
7. [Bash Automation Scripts](#7-bash-automation-scripts)
8. [Troubleshooting Scenarios](#8-troubleshooting-scenarios)
 
---
 
## 1. VM Setup
 
Download and install Oracle VirtualBox and the Extension Pack from [virtualbox.org](https://www.virtualbox.org/wiki/Downloads).
 
Download Ubuntu 24.04.2 LTS ISO from [ubuntu.com](https://ubuntu.com/download/desktop).
 
Create a new VM in VirtualBox:
 
- Name: `LinuxVM`
- Type: Linux / Ubuntu (64-bit)
- RAM: 2048 MB minimum
- CPU: 2 cores
- Disk: 25 GB (dynamically allocated)
 
Mount the ISO, complete the installation, and set hostname to `LinuxVM`.
 
After install, update the system:
 
```bash
sudo apt update && sudo apt upgrade -y
```
 
Install essential tools:
 
```bash
sudo apt install git curl wget net-tools tree ufw fail2ban -y
```
 
---
 
## 2. User & Group Management
 
### Create Groups
 
```bash
sudo groupadd sysadmins
sudo groupadd developers
sudo groupadd auditors
```
 
### Create Users and Assign Groups
 
```bash
# Create users
sudo useradd -m -s /bin/bash -G sysadmins alice
sudo useradd -m -s /bin/bash -G developers bob
sudo useradd -m -s /bin/bash -G auditors carol
 
# Set passwords
sudo passwd alice
sudo passwd bob
sudo passwd carol
```
 
### Set Password Aging Policies
 
Enforce password expiration and minimum age to align with security compliance standards:
 
```bash
sudo chage -M 90 -m 7 -W 14 alice
sudo chage -M 90 -m 7 -W 14 bob
sudo chage -M 90 -m 7 -W 14 carol
```
 
### Verify User Configuration
 
```bash
# View user details
id alice
cat /etc/passwd | grep alice
 
# View group membership
groups alice
cat /etc/group | grep sysadmins
 
# View password aging policy
sudo chage -l alice
```
 
### Lock and Unlock Accounts
 
```bash
# Lock account
sudo usermod -L bob
 
# Unlock account
sudo usermod -U bob
 
# Verify lock status
sudo passwd -S bob
```
 
---
 
## 3. SSH Hardening
 
### Generate SSH Key Pair (on host machine)
 
```bash
ssh-keygen -t ed25519 -C "lab-key"
```
 
### Copy Public Key to VM
 
```bash
ssh-copy-id user@<VM_IP>
```
 
### Harden SSH Configuration
 
Edit the SSH daemon config:
 
```bash
sudo vim /etc/ssh/sshd_config
```
 
Apply these settings:
 
```
# Disable root login
PermitRootLogin no
 
# Disable password authentication (key-based only)
PasswordAuthentication no
 
# Limit authentication attempts
MaxAuthTries 3
 
# Set idle timeout (5 minutes)
ClientAliveInterval 300
ClientAliveCountMax 0
 
# Change default port
Port 2222
 
# Allow only specific users
AllowUsers alice
```
 
Restart SSH and verify:
 
```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```
 
### Configure fail2ban
 
fail2ban monitors logs and bans IPs with repeated failed login attempts:
 
```bash
sudo vim /etc/fail2ban/jail.local
```
 
```
[sshd]
enabled = true
port = 2222
maxretry = 3
bantime = 3600
findtime = 600
```
 
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
 
# Check ban status
sudo fail2ban-client status sshd
```
 
---
 
## 4. Sudo Policy Configuration
 
### Edit Sudoers File Safely
 
Always use `visudo` to prevent syntax errors that could lock you out:
 
```bash
sudo visudo
```
 
### Configure Group-Based Sudo Policies
 
```bash
# sysadmins - full sudo access
%sysadmins ALL=(ALL:ALL) ALL
 
# developers - can restart application services only
%developers ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp.service
 
# auditors - read-only commands only
%auditors ALL=(ALL) NOPASSWD: /bin/cat, /bin/ls, /usr/bin/journalctl
```
 
### Create a Drop-In Sudoers File
 
Rather than editing the main sudoers file directly, use drop-in files for cleaner management:
 
```bash
sudo visudo -f /etc/sudoers.d/sysadmins
sudo visudo -f /etc/sudoers.d/developers
sudo visudo -f /etc/sudoers.d/auditors
```
 
### Verify Sudo Permissions
 
```bash
# Test as bob (developer)
sudo -u bob sudo systemctl restart myapp.service
 
# View effective sudo rules for a user
sudo -l -U bob
```
 
---
 
## 5. Firewall Configuration with firewalld
 
### Install and Enable firewalld
 
```bash
sudo apt install firewalld -y
sudo systemctl enable firewalld
sudo systemctl start firewalld
```
 
### View Current Configuration
 
```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --get-active-zones
```
 
### Configure Zones and Rules
 
```bash
# Allow SSH on custom port (permanent)
sudo firewall-cmd --permanent --add-port=2222/tcp
 
# Allow HTTP and HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
 
# Remove default SSH port (no longer needed)
sudo firewall-cmd --permanent --remove-service=ssh
 
# Reload to apply changes
sudo firewall-cmd --reload
 
# Verify
sudo firewall-cmd --list-all
```
 
### Block a Specific IP (simulating threat response)
 
```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" reject'
sudo firewall-cmd --reload
```
 
---
 
## 6. Systemd Service Management
 
### Create a Custom Service
 
This simulates managing an application service the way a sysadmin would in production:
 
```bash
sudo vim /etc/systemd/system/myapp.service
```
 
```ini
[Unit]
Description=My Application Service
After=network.target
 
[Service]
Type=simple
User=bob
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
 
[Install]
WantedBy=multi-user.target
```
 
Create a placeholder app:
 
```bash
sudo mkdir -p /opt/myapp
echo 'import time; print("App running"); time.sleep(3600)' | sudo tee /opt/myapp/app.py
```
 
### Enable and Manage the Service
 
```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
sudo systemctl status myapp.service
```
 
### Simulate a Service Failure and Recovery
 
```bash
# Force kill the service
sudo systemctl kill myapp.service
 
# Watch systemd auto-restart it
sudo systemctl status myapp.service
 
# View restart events in journal
sudo journalctl -u myapp.service -n 20
```
 
---
 
## 7. Bash Automation Scripts
 
### Backup Script
 
Backs up specified directories, timestamps the archive, and enforces a 7-day retention policy:
 
```bash
vim ~/backup.sh
```
 
```bash
#!/bin/bash
 
DATE=$(date +"%Y-%m-%d_%H-%M-%S")
HOSTNAME=$(hostname)
FILENAME="backup_${HOSTNAME}_${DATE}.tar.gz"
BACKUP_DIR="$HOME/backups"
RETENTION_DAYS=7
LOG_TAG="vm-backup"
 
SPECIFIC_DIRS=(
  "$HOME/Documents"
  "$HOME/Desktop"
  "$HOME/.config"
  "$HOME/projects"
  "/etc"
)
 
USER_HOMES=($(awk -F: '$3 >= 1000 && $7 ~ /bash|sh/ { print $6 }' /etc/passwd))
SOURCE_DIRS=("${SPECIFIC_DIRS[@]}" "${USER_HOMES[@]}")
 
mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/$FILENAME" "${SOURCE_DIRS[@]}"
logger -t "$LOG_TAG" "Backup created: $BACKUP_DIR/$FILENAME"
 
find "$BACKUP_DIR" -name "*.tar.gz" -type f -mtime +$RETENTION_DAYS -exec rm {} \;
logger -t "$LOG_TAG" "Old backups older than $RETENTION_DAYS days deleted"
 
echo "Backup complete: $FILENAME"
```
 
### System Health Check Script
 
Monitors CPU, memory, and disk usage and logs alerts when thresholds are exceeded:
 
```bash
vim ~/health_check.sh
```
 
```bash
#!/bin/bash
 
LOG_FILE="/var/log/health_check.log"
DATE=$(date +"%Y-%m-%d %H:%M:%S")
 
CPU_THRESHOLD=80
MEM_THRESHOLD=80
DISK_THRESHOLD=85
 
# CPU usage
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)
 
# Memory usage
MEM_USAGE=$(free | awk '/Mem:/ {printf "%.0f", $3/$2 * 100}')
 
# Disk usage (root partition)
DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
 
echo "[$DATE] CPU: ${CPU_USAGE}% | MEM: ${MEM_USAGE}% | DISK: ${DISK_USAGE}%" >> "$LOG_FILE"
 
if [ "$CPU_USAGE" -gt "$CPU_THRESHOLD" ]; then
  echo "[$DATE] ALERT: CPU usage at ${CPU_USAGE}%" >> "$LOG_FILE"
  logger -t "health-check" "ALERT: CPU usage at ${CPU_USAGE}%"
fi
 
if [ "$MEM_USAGE" -gt "$MEM_THRESHOLD" ]; then
  echo "[$DATE] ALERT: Memory usage at ${MEM_USAGE}%" >> "$LOG_FILE"
  logger -t "health-check" "ALERT: Memory usage at ${MEM_USAGE}%"
fi
 
if [ "$DISK_USAGE" -gt "$DISK_THRESHOLD" ]; then
  echo "[$DATE] ALERT: Disk usage at ${DISK_USAGE}%" >> "$LOG_FILE"
  logger -t "health-check" "ALERT: Disk usage at ${DISK_USAGE}%"
fi
```
 
### Log Rotation Script
 
Compresses and archives logs older than 7 days:
 
```bash
vim ~/log_rotate.sh
```
 
```bash
#!/bin/bash
 
LOG_DIR="/var/log"
ARCHIVE_DIR="$HOME/log_archives"
RETENTION_DAYS=7
DATE=$(date +"%Y-%m-%d")
 
mkdir -p "$ARCHIVE_DIR"
 
find "$LOG_DIR" -name "*.log" -mtime +$RETENTION_DAYS | while read logfile; do
  BASENAME=$(basename "$logfile")
  gzip -c "$logfile" > "$ARCHIVE_DIR/${BASENAME}_${DATE}.gz"
  echo "Archived: $logfile"
  logger -t "log-rotate" "Archived: $logfile"
done
 
find "$ARCHIVE_DIR" -name "*.gz" -mtime +30 -exec rm {} \;
logger -t "log-rotate" "Removed archives older than 30 days"
```
 
### Make Scripts Executable and Schedule with Cron
 
```bash
chmod +x ~/backup.sh ~/health_check.sh ~/log_rotate.sh
 
crontab -e
```
 
```
# Backup daily at 1:00 AM
0 1 * * * /home/USERNAME/backup.sh
 
# Health check every 15 minutes
*/15 * * * * /home/USERNAME/health_check.sh
 
# Log rotation weekly on Sunday at 2:00 AM
0 2 * * 0 /home/USERNAME/log_rotate.sh
```
 
---
 
## 8. Troubleshooting Scenarios
 
This section documents real failure scenarios, how they were diagnosed, and how they were resolved — simulating production incident response.
 
---
 
### Scenario 1 — Service Fails to Start
 
**Symptom:** `sudo systemctl start myapp.service` fails with no clear error.
 
**Diagnosis:**
 
```bash
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -n 50
sudo journalctl -xe
```
 
**Root Cause:** Script path in ExecStart was incorrect or the app file had a permission issue.
 
**Resolution:**
 
```bash
# Verify the file exists and is executable
ls -l /opt/myapp/app.py
 
# Fix permissions
sudo chmod 755 /opt/myapp/app.py
sudo chown bob:bob /opt/myapp/app.py
 
# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```
 
---
 
### Scenario 2 — SSH Login Denied After Config Change
 
**Symptom:** Cannot SSH into the VM after editing `sshd_config`.
 
**Diagnosis:**
 
```bash
# Check SSH service status from local console
sudo systemctl status sshd
 
# Look for config errors
sudo sshd -t
```
 
**Root Cause:** Syntax error in `sshd_config` or wrong port specified in the SSH client.
 
**Resolution:**
 
```bash
# Fix the config error identified by sshd -t
sudo vim /etc/ssh/sshd_config
 
# Restart SSH
sudo systemctl restart sshd
 
# Connect specifying correct port
ssh -p 2222 alice@<VM_IP>
```
 
---
 
### Scenario 3 — Disk Space Exhaustion
 
**Symptom:** Services begin failing. Logs stop writing.
 
**Diagnosis:**
 
```bash
# Check disk usage
df -h
 
# Find large files and directories
du -sh /* 2>/dev/null | sort -rh | head -20
du -sh /var/log/* | sort -rh | head -10
```
 
**Root Cause:** Log files grew unchecked, filling the root partition.
 
**Resolution:**
 
```bash
# Clear old journal logs
sudo journalctl --vacuum-time=7d
 
# Remove old log archives
sudo find /var/log -name "*.gz" -mtime +7 -exec rm {} \;
 
# Verify disk space recovered
df -h
```
 
---
 
### Scenario 4 — Permission Denied on Script Execution
 
**Symptom:** Running `./backup.sh` returns `Permission denied`.
 
**Diagnosis:**
 
```bash
ls -l ~/backup.sh
```
 
**Root Cause:** Script is not marked executable.
 
**Resolution:**
 
```bash
chmod +x ~/backup.sh
./backup.sh
```
 
---
 
### Scenario 5 — fail2ban Blocking Legitimate User
 
**Symptom:** A user is locked out after too many failed SSH attempts.
 
**Diagnosis:**
 
```bash
sudo fail2ban-client status sshd
```
 
**Resolution:**
 
```bash
# Unban the IP
sudo fail2ban-client set sshd unbanip <IP_ADDRESS>
 
# Verify
sudo fail2ban-client status sshd
```
 
---
 
## Tools Used
 
| Tool | Purpose |
|---|---|
| journalctl | Service and system log analysis |
| systemctl | Service management and status |
| ss | Active connection and socket inspection |
| top / htop | Real-time process and CPU/memory monitoring |
| df / du | Disk usage analysis |
| firewall-cmd | Firewall rule management |
| fail2ban-client | Intrusion prevention status and management |
| chage | Password aging policy enforcement |
| visudo | Safe sudoers policy editing |
 
---
 
[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:kennymiranda000@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/kenneth-miranda-xyz)
