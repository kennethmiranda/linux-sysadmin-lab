# Linux Systems Administration & Automation Lab

A hands-on, multi-node Linux lab built on VirtualBox simulating enterprise 
server operations across both RHEL and Ubuntu environments. This lab covers 
user and group management, SSH hardening, sudo policy enforcement, firewall 
configuration, systemd service management, Bash automation, and real-world 
troubleshooting — and served as the primary preparation environment for the 
Red Hat Certified System Administrator (RHCSA) exam, which I passed in 
April 2026.

---

## Lab Environment

| Component       | Node 1 — RHEL          | Node 2 — Ubuntu         |
|-----------------|------------------------|-------------------------|
| OS              | RHEL 9.4               | Ubuntu 24.04.2 LTS      |
| Hostname        | rhel-node              | ubuntu-node             |
| Virtualization  | Oracle VirtualBox      | Oracle VirtualBox       |
| Host OS         | Windows 11             | Windows 11              |
| Resources       | 2 vCPUs, 2GB RAM, 25GB | 2 vCPUs, 2GB RAM, 25GB  |
| Primary Role    | RHCSA exam practice,   | Secondary admin node,   |
|                 | firewalld, systemd,    | Bash automation,        |
|                 | SELinux, user mgmt     | cross-node SSH, backups |

Both VMs run on an internal VirtualBox network, allowing SSH between nodes 
and simulating a basic multi-server environment.

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
9. [RHCSA Exam Practice](#9-rhcsa-exam-practice)

---

## 1. VM Setup

### VirtualBox Network Configuration

Both VMs are attached to the same Internal Network in VirtualBox, 
enabling direct communication between nodes:

- VM1 (rhel-node): Internal Network `labnet`, static IP `192.168.56.10`
- VM2 (ubuntu-node): Internal Network `labnet`, static IP `192.168.56.20`

### RHEL Node Setup

Download RHEL 9 ISO via Red Hat's free Developer Program at 
[developers.redhat.com](https://developers.redhat.com).

Create a new VM in VirtualBox:

- Name: `rhel-node`
- Type: Linux / Red Hat (64-bit)
- RAM: 2048 MB | CPU: 2 cores | Disk: 25 GB (dynamically allocated)

After installation, register with Red Hat and update:

```bash
sudo subscription-manager register --username  --auto-attach
sudo dnf update -y
```

Install essential tools:

```bash
sudo dnf install git curl wget net-tools tree firewalld -y
```

Set static IP and hostname:

```bash
sudo hostnamectl set-hostname rhel-node
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.56.10/24 \
  ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

### Ubuntu Node Setup

Download Ubuntu 24.04.2 LTS ISO from [ubuntu.com](https://ubuntu.com/download/desktop).

Create a new VM in VirtualBox:

- Name: `ubuntu-node`
- Type: Linux / Ubuntu (64-bit)
- RAM: 2048 MB | CPU: 2 cores | Disk: 25 GB (dynamically allocated)

After installation, update and install tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl wget net-tools tree ufw fail2ban -y
```

Set static IP and hostname:

```bash
sudo hostnamectl set-hostname ubuntu-node
```

Edit `/etc/netplan/01-netcfg.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [192.168.56.20/24]
      nameservers:
        addresses: [8.8.8.8]
```

```bash
sudo netplan apply
```

### Verify Node Connectivity

```bash
# From rhel-node
ping -c 3 192.168.56.20

# From ubuntu-node
ping -c 3 192.168.56.10
```

---

## 2. User & Group Management

Performed on both nodes to practice across distributions. RHEL uses 
`useradd`/`groupadd` identically to Ubuntu; differences in defaults 
(e.g., home directory creation) are noted where relevant.

### Create Groups

```bash
sudo groupadd sysadmins
sudo groupadd developers
sudo groupadd auditors
```

### Create Users and Assign Groups

```bash
sudo useradd -m -s /bin/bash -G sysadmins alice
sudo useradd -m -s /bin/bash -G developers bob
sudo useradd -m -s /bin/bash -G auditors carol

sudo passwd alice
sudo passwd bob
sudo passwd carol
```

> On RHEL, `useradd` does not create a home directory by default unless 
> `-m` is explicitly passed — this is a common RHCSA exam gotcha.

### Set Password Aging Policies

```bash
sudo chage -M 90 -m 7 -W 14 alice
sudo chage -M 90 -m 7 -W 14 bob
sudo chage -M 90 -m 7 -W 14 carol
```

### Verify User Configuration

```bash
id alice
cat /etc/passwd | grep alice
groups alice
cat /etc/group | grep sysadmins
sudo chage -l alice
```

### Lock and Unlock Accounts

```bash
sudo usermod -L bob
sudo usermod -U bob
sudo passwd -S bob
```

---

## 3. SSH Hardening

SSH was hardened on both nodes. Cross-node SSH (rhel-node ↔ ubuntu-node) 
was also configured to simulate a real multi-server environment where 
admins move between systems using key-based auth.

### Generate SSH Key Pair

```bash
# On host machine or rhel-node
ssh-keygen -t ed25519 -C "lab-key"
```

### Copy Public Key to Both Nodes

```bash
ssh-copy-id user@192.168.56.10   # rhel-node
ssh-copy-id user@192.168.56.20   # ubuntu-node
```

### Harden SSH Configuration (applied to both nodes)

```bash
sudo vim /etc/ssh/sshd_config
```

```
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 0
Port 2222
AllowUsers alice
```

```bash
# RHEL
sudo systemctl restart sshd

# Ubuntu
sudo systemctl restart ssh
```

> On RHEL with SELinux enforcing, changing the SSH port requires updating 
> SELinux port policy before the service will bind to the new port:

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload
```

### Configure fail2ban (Ubuntu Node)

```bash
sudo vim /etc/fail2ban/jail.local
```

```ini
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

### Create Drop-In Sudoers Files

```bash
sudo visudo -f /etc/sudoers.d/sysadmins
sudo visudo -f /etc/sudoers.d/developers
sudo visudo -f /etc/sudoers.d/auditors
```

### Verify Sudo Permissions

```bash
sudo -l -U bob
sudo -u bob sudo systemctl restart myapp.service
```

---

## 5. Firewall Configuration with firewalld

`firewalld` is the default firewall manager on RHEL and was also installed 
on the Ubuntu node to practice a consistent toolset across both systems.

### Install and Enable firewalld

```bash
# RHEL (pre-installed, ensure it's running)
sudo systemctl enable firewalld
sudo systemctl start firewalld

# Ubuntu
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
# Allow SSH on custom port
sudo firewall-cmd --permanent --add-port=2222/tcp

# Allow HTTP and HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Remove default SSH port
sudo firewall-cmd --permanent --remove-service=ssh

sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### Block a Specific IP (simulating threat response)

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" \
  source address="192.168.1.100" reject'
sudo firewall-cmd --reload
```

---

## 6. Systemd Service Management

Custom services were created and managed on the RHEL node as part of 
RHCSA exam preparation, where systemd service configuration is a core 
tested objective.

### Create a Custom Service

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
echo 'import time; print("App running"); time.sleep(3600)' | \
  sudo tee /opt/myapp/app.py
sudo chown bob:bob /opt/myapp/app.py
sudo chmod 755 /opt/myapp/app.py
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
sudo systemctl kill myapp.service
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -n 20
```

---

## 7. Bash Automation Scripts

Scripts were developed on the Ubuntu node and structured to reflect 
enterprise operational runbook practices — including timestamped logging, 
configurable retention policies, and syslog integration via `logger`. 
The backup script was extended to pull from both nodes over SSH.

### Backup Script

Backs up specified directories, timestamps the archive, and enforces a 
7-day retention policy. Pulls a remote archive from rhel-node via SSH:

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
REMOTE_USER="alice"
REMOTE_HOST="192.168.56.10"
REMOTE_BACKUP_DIR="/home/alice/remote_backups"

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

# Pull backup archive from rhel-node
ssh "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p ${REMOTE_BACKUP_DIR}"
scp "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BACKUP_DIR}/*.tar.gz" \
  "$BACKUP_DIR/" 2>/dev/null
logger -t "$LOG_TAG" "Remote backup pulled from ${REMOTE_HOST}"

find "$BACKUP_DIR" -name "*.tar.gz" -type f -mtime +$RETENTION_DAYS -exec rm {} \;
logger -t "$LOG_TAG" "Old backups older than $RETENTION_DAYS days deleted"

echo "Backup complete: $FILENAME"
```

### System Health Check Script

Monitors CPU, memory, and disk usage across both nodes and logs alerts 
when thresholds are exceeded:

```bash
vim ~/health_check.sh
```

```bash
#!/bin/bash

LOG_FILE="/var/log/health_check.log"
DATE=$(date +"%Y-%m-%d %H:%M:%S")
REMOTE_USER="alice"
REMOTE_HOST="192.168.56.10"

CPU_THRESHOLD=80
MEM_THRESHOLD=80
DISK_THRESHOLD=85

check_node() {
  local label="$1"
  local cpu mem disk

  if [ "$label" = "local" ]; then
    cpu=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)
    mem=$(free | awk '/Mem:/ {printf "%.0f", $3/$2 * 100}')
    disk=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
  else
    cpu=$(ssh "${REMOTE_USER}@${REMOTE_HOST}" \
      "top -bn1 | grep 'Cpu(s)' | awk '{print \$2}' | cut -d. -f1")
    mem=$(ssh "${REMOTE_USER}@${REMOTE_HOST}" \
      "free | awk '/Mem:/ {printf \"%.0f\", \$3/\$2 * 100}'")
    disk=$(ssh "${REMOTE_USER}@${REMOTE_HOST}" \
      "df / | awk 'NR==2 {print \$5}' | tr -d '%'")
  fi

  echo "[$DATE][$label] CPU: ${cpu}% | MEM: ${mem}% | DISK: ${disk}%" \
    >> "$LOG_FILE"

  [ "$cpu" -gt "$CPU_THRESHOLD" ] && \
    logger -t "health-check" "ALERT [$label]: CPU at ${cpu}%"
  [ "$mem" -gt "$MEM_THRESHOLD" ] && \
    logger -t "health-check" "ALERT [$label]: MEM at ${mem}%"
  [ "$disk" -gt "$DISK_THRESHOLD" ] && \
    logger -t "health-check" "ALERT [$label]: DISK at ${disk}%"
}

check_node "ubuntu-node"
check_node "rhel-node"
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

This section documents real failure scenarios encountered during the lab, 
how they were diagnosed, and how they were resolved — simulating production 
incident response. Scenarios marked **[RHEL]** were reproduced specifically 
on the RHEL node as part of RHCSA exam preparation.

---

### Scenario 1 — Service Fails to Start [RHEL]

**Symptom:** `sudo systemctl start myapp.service` fails with no clear error.

**Diagnosis:**

```bash
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -n 50
sudo journalctl -xe
```

**Root Cause:** Script path in ExecStart was incorrect or the app file 
had a permission issue. On RHEL, SELinux context mismatches can also 
block service execution even when file permissions appear correct.

**Resolution:**

```bash
ls -l /opt/myapp/app.py
sudo chmod 755 /opt/myapp/app.py
sudo chown bob:bob /opt/myapp/app.py

# Check for SELinux denials (RHEL)
sudo ausearch -m avc -ts recent
sudo restorecon -v /opt/myapp/app.py

sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

---

### Scenario 2 — SSH Login Denied After Config Change

**Symptom:** Cannot SSH into a node after editing `sshd_config`.

**Diagnosis:**

```bash
sudo systemctl status sshd
sudo sshd -t
```

**Root Cause:** Syntax error in `sshd_config` or, on RHEL, SELinux 
blocking the new port after a port change without updating policy.

**Resolution:**

```bash
# Fix config syntax
sudo vim /etc/ssh/sshd_config

# On RHEL — update SELinux port policy if port was changed
sudo semanage port -a -t ssh_port_t -p tcp 2222

sudo systemctl restart sshd
ssh -p 2222 alice@192.168.56.10
```

---

### Scenario 3 — Disk Space Exhaustion

**Symptom:** Services begin failing. Logs stop writing.

**Diagnosis:**

```bash
df -h
du -sh /* 2>/dev/null | sort -rh | head -20
du -sh /var/log/* | sort -rh | head -10

# Identify specific large files using strace on a failing write
sudo strace -e trace=write df -h 2>&1 | grep -i "No space"
```

**Root Cause:** Log files grew unchecked, filling the root partition.

**Resolution:**

```bash
sudo journalctl --vacuum-time=7d
sudo find /var/log -name "*.gz" -mtime +7 -exec rm {} \;
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
sudo fail2ban-client set sshd unbanip <IP_ADDRESS>
sudo fail2ban-client status sshd
```

---

### Scenario 6 — SELinux Blocking Service [RHEL]

**Symptom:** A service starts successfully but cannot write to its log 
directory despite correct file permissions.

**Diagnosis:**

```bash
# Check for AVC denials in the audit log
sudo ausearch -m avc -ts recent
sudo cat /var/log/audit/audit.log | grep denied
```

**Root Cause:** The service's target directory had the wrong SELinux 
file context, causing enforcing mode to block writes.

**Resolution:**

```bash
# View current context
ls -Z /opt/myapp/

# Apply correct context
sudo semanage fcontext -a -t var_log_t "/opt/myapp/logs(/.*)?"
sudo restorecon -Rv /opt/myapp/logs/

# Verify
ls -Z /opt/myapp/logs/
```

---

## 9. RHCSA Exam Practice

This lab served as the primary hands-on preparation environment for the 
Red Hat Certified System Administrator (RHCSA) exam, which I passed in 
April 2026. All tasks below were practiced on the RHEL node (`rhel-node`) 
and reflect real exam-style objectives from the EX200 exam.

---

### Task 1 — Create Users with Supplementary Groups

**Objective:** Create user `john` with UID 1050, primary group `developers`, 
and supplementary group `sysadmins`.

```bash
sudo groupadd -g 2001 developers
sudo groupadd -g 2002 sysadmins
sudo useradd -u 1050 -g developers -G sysadmins -m -s /bin/bash john
id john
```

**Verify:** `id john` should show UID 1050, primary group developers, 
supplementary group sysadmins.

---

### Task 2 — Configure a Cron Job for a Specific User

**Objective:** Schedule a script to run as user `alice` every day at 3:30 AM.

```bash
sudo crontab -u alice -e
```
```
30 3 * * * /home/alice/backup.sh
```

```bash
sudo crontab -u alice -l
```

---

### Task 3 — Set File Permissions and ACLs

**Objective:** Create `/data/shared` where `developers` can read and write, 
others have no access. Extend with read-only access for `auditors` via ACL.

```bash
sudo mkdir -p /data/shared
sudo chown root:developers /data/shared
sudo chmod 770 /data/shared

ls -ld /data/shared
```

```bash
sudo dnf install acl -y    # RHEL
sudo setfacl -m g:auditors:r-x /data/shared
getfacl /data/shared
```

---

### Task 4 — Configure a Systemd Service to Start on Boot

**Objective:** Ensure `myapp.service` starts automatically after reboot.

```bash
sudo systemctl enable myapp.service
sudo systemctl reboot
# After reboot:
sudo systemctl status myapp.service
```

---

### Task 5 — Diagnose and Fix a Failed Service

**Objective:** A service is failing on startup. Identify the cause and 
restore it to running state.

```bash
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -n 30
sudo ausearch -m avc -ts recent    # check for SELinux denials
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

---

### Task 6 — Manage Password Policies

**Objective:** Configure user `bob` so their password expires every 60 days 
with a 7-day warning.

```bash
sudo chage -M 60 -W 7 bob
sudo chage -l bob
```

---

### Task 7 — Configure SELinux Modes and Contexts

**Objective:** Verify SELinux is enforcing and correct a file context to 
allow a service to access it.

```bash
# Check current mode
getenforce
sestatus

# Temporarily set to permissive (for testing only)
sudo setenforce 0

# Set back to enforcing
sudo setenforce 1

# Make enforcing persistent across reboots
sudo vim /etc/selinux/config
# Set: SELINUX=enforcing

# Fix a file context
sudo restorecon -Rv /opt/myapp/
```

---

### Task 8 — Configure a Network Interface with nmcli

**Objective:** Assign a static IP to the primary interface without a 
graphical tool.

```bash
sudo nmcli con show
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 192.168.56.10/24 \
  ipv4.gateway 192.168.56.1 \
  ipv4.dns 8.8.8.8 \
  ipv4.method manual
sudo nmcli con up "Wired connection 1"

ip a
```

---

## Tools Used

| Tool              | Purpose                                              |
|-------------------|------------------------------------------------------|
| journalctl        | Service and system log analysis                      |
| systemctl         | Service management and status                        |
| ss                | Active connection and socket inspection              |
| top / htop        | Real-time process and CPU/memory monitoring          |
| df / du           | Disk usage analysis                                  |
| strace            | System call tracing for permission and I/O debugging |
| firewall-cmd      | Firewall rule management                             |
| fail2ban-client   | Intrusion prevention status and management           |
| chage             | Password aging policy enforcement                    |
| visudo            | Safe sudoers policy editing                          |
| ausearch          | SELinux audit log analysis                           |
| semanage          | SELinux policy management (ports, file contexts)     |
| restorecon        | Restore default SELinux file contexts                |
| nmcli             | Network interface configuration (RHEL)               |
| /var/log analysis | Manual log inspection across both nodes              |
 
---
 
[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:kennymiranda000@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/kenneth-miranda-xyz)
