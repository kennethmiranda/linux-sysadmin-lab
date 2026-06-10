# Linux Systems Administration & Automation Lab

A hands-on, multi-node Linux lab built on VirtualBox simulating enterprise server
operations across RHEL and Ubuntu environments. This lab covers user and group
management, SSH hardening, sudo policy enforcement, firewall configuration, systemd
service management, centralized logging, infrastructure monitoring, container
management, configuration management automation, and real-world troubleshooting.
It served as the primary preparation environment for the Red Hat Certified System
Administrator (RHCSA) exam, passed in April 2026.

---

## Lab Environment

| Component    | Node 1 — RHEL                                                                  | Node 2 — Ubuntu                                                                 |
|--------------|--------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| OS           | RHEL 9.8                                                                       | Ubuntu 26.04 LTS                                                                |
| Hostname     | rhel-node                                                                      | ubuntu-node                                                                     |
| Virtualization | Oracle VirtualBox 7.2.8                                                      | Oracle VirtualBox 7.2.8                                                         |
| Host OS      | Windows 11                                                                     | Windows 11                                                                      |
| Resources    | 2 vCPUs, 2 GB RAM, 25 GB                                                       | 2 vCPUs, 2 GB RAM, 25 GB                                                        |
| Primary Role | RHCSA practice, firewalld, systemd, SELinux, user mgmt, centralized log server | Bash automation, monitoring stack, Docker, Ansible control node, log forwarding |

---

## Network Architecture

Both VMs use an Internal Network (`labnet`) for node-to-node communication with
static IPs. A separate NAT adapter provides internet access for package downloads.

| Adapter  | Type             | Purpose                        |
|----------|------------------|--------------------------------|
| enp0s3   | Internal Network | Lab communication (192.168.56.x)|
| enp0s8   | NAT              | Internet access for downloads  |

**Why Internal Network over Bridged or Host-Only:**

Internal Network was chosen to isolate the lab environment from the physical network,
preventing any unintended traffic. Bridged networking would expose VMs to the home
network and could cause IP conflicts. Host-Only networking was not available in this
VirtualBox installation.

**Tradeoff:** Internal Network means the host machine cannot directly reach the VMs.
Browser access to Prometheus and Grafana required an SSH tunnel workaround — see
[Issue 8](#issue-8----host-browser-cannot-reach-prometheus-or-grafana) for details.

---

## Table of Contents

1. [VM Setup](#1-vm-setup)
2. [User & Group Management](#2-user--group-management)
3. [SSH Hardening](#3-ssh-hardening)
4. [Sudo Policy Configuration](#4-sudo-policy-configuration)
5. [Firewall Configuration with firewalld](#5-firewall-configuration-with-firewalld)
6. [Systemd Service Management](#6-systemd-service-management)
7. [Centralized Logging with rsyslog](#7-centralized-logging-with-rsyslog)
8. [Monitoring with Prometheus & Grafana](#8-monitoring-with-prometheus--grafana)
9. [Container Management with Docker](#9-container-management-with-docker)
10. [Configuration Management with Ansible](#10-configuration-management-with-ansible)
11. [Bash Automation Scripts](#11-bash-automation-scripts)
12. [Troubleshooting Scenarios](#12-troubleshooting-scenarios)
13. [RHCSA Exam Practice](#13-rhcsa-exam-practice)
14. [Issues & Lessons Learned](#14-issues--lessons-learned)

---

## 1. VM Setup

### VirtualBox Network Configuration

Both VMs are attached to the same Internal Network in VirtualBox:

- VM1 (rhel-node): Internal Network `labnet`, static IP `192.168.56.10`
- VM2 (ubuntu-node): Internal Network `labnet`, static IP `192.168.56.20`

A second NAT adapter is added to each VM for internet access during package
installation. The NAT adapter on RHEL requires manual nmcli connection creation —
see [Issue 1](#issue-1----rhel-nat-adapter-requires-manual-nmcli-connection-creation).

### RHEL Node Setup

Download RHEL 9 ISO via Red Hat's free Developer Program at
[developers.redhat.com](https://developers.redhat.com).

> **Version note:** Use the DVD ISO, not the Boot ISO. The DVD ISO is self-contained
> and does not require internet during installation. The Boot ISO requires network
> access to download packages mid-install which is unreliable in a lab environment.
> Use the x86_64 architecture version.

After installation, register and update:

```bash
sudo subscription-manager register --username <username> --auto-attach
sudo dnf update -y
sudo dnf install git curl wget net-tools tree firewalld -y
```

Set static IP and hostname:

```bash
sudo hostnamectl set-hostname rhel-node

# Note: Interface may be named enp0s3 instead of "Wired connection 1"
# Run: ip a  to confirm your interface name before running nmcli
sudo nmcli con mod "enp0s3" ipv4.addresses 192.168.56.10/24 ipv4.method manual
sudo nmcli con up "enp0s3"
```

> **Interface naming note:** The interface name varies by VM configuration.
> Run `ip a` first to identify the correct name before running nmcli commands.
> In this lab the interface was `enp0s3` rather than "Wired connection 1".

### Ubuntu Node Setup

Download Ubuntu 26.04 LTS Server ISO from [ubuntu.com](https://ubuntu.com/download/server).

> **Version note:** Ubuntu 26.04 LTS works identically to 24.04 for all lab
> commands. Use Server ISO not Desktop — Server has lower memory overhead and
> is more representative of enterprise Linux deployments.

After installation:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl wget net-tools tree ufw fail2ban -y
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

![rhel-node ip a and successful ping to ubuntu-node](screenshots/2%20rhel%20node%20ip%20a%20and%20ping.PNG)
![ubuntu-node ip a and successful ping to rhel-node](screenshots/3%20ubuntu%20node%20ip%20a%20and%20ping.PNG)

---

## 2. User & Group Management

Performed on both nodes. RHEL and Ubuntu use identical `useradd`/`groupadd` syntax.

> **Important:** The primary user account (alice) is created during OS installation
> and already exists. Do not attempt to recreate it with useradd — add it to groups
> using usermod instead. Only bob and carol need to be created fresh.

### Create Groups

```bash
sudo groupadd sysadmins
sudo groupadd developers
sudo groupadd auditors
```

### Create and Assign Users

```bash
# alice already exists from OS installation — add to sysadmins group only
sudo usermod -aG sysadmins alice

# bob and carol do not exist — create them normally
sudo useradd -m -s /bin/bash -G developers bob
sudo useradd -m -s /bin/bash -G auditors carol

sudo passwd bob
sudo passwd carol
```

> **RHEL note:** `useradd` on RHEL does not create a home directory by default
> unless `-m` is explicitly passed. Always include `-m` on RHEL.

### Set Password Aging Policies

```bash
sudo chage -M 90 -m 7 -W 14 alice
sudo chage -M 90 -m 7 -W 14 bob
sudo chage -M 90 -m 7 -W 14 carol
```

### Verify User Configuration

```bash
id alice
groups alice
sudo chage -l alice
```

![id and chage -l on rhel-node confirming group membership and password policy](screenshots/4%20id%20and%20chage%20-l.PNG)
![id and chage -l on ubuntu-node](screenshots/4-2%20id%20and%20chage%20-l.PNG)

### Lock and Unlock Accounts

```bash
sudo usermod -L bob
sudo passwd -S bob
sudo usermod -U bob
sudo passwd -S bob
```

![Lock and unlock bob account with status verification](screenshots/4-3%20lock%20unlock%20bob.PNG)

---

## 3. SSH Hardening

SSH was hardened on both nodes with key-based authentication and a custom port.
Cross-node SSH was configured to simulate a real multi-server environment.

### Generate SSH Key Pair

```bash
ssh-keygen -t ed25519 -C "lab-key"
```

### Copy Public Keys Between Nodes

```bash
# From rhel-node to ubuntu-node
ssh-copy-id -p 22 alice@192.168.56.20

# Add ubuntu-node's public key to rhel-node authorized_keys manually
# On ubuntu-node:
cat ~/.ssh/id_ed25519.pub
# Copy output, then on rhel-node:
echo "paste-ubuntu-public-key-here" >> ~/.ssh/authorized_keys
```

> Also add your host machine's public key to both nodes' authorized_keys
> to enable SSH tunnel access from the host browser later.

### Harden SSH Configuration

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

> **RHEL SELinux note:** Changing the SSH port requires updating SELinux port
> policy before the service will bind to the new port:

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload
```

![rhel-node SSH into ubuntu-node on port 2222](screenshots/5-1%20rhel%20ssh%20to%20ubuntu.PNG)
![RHEL SELinux port policy update and firewall rule](screenshots/5-2%20rhel%20sshd%20selinux%20firewall.PNG)

> **Ubuntu cloud-init override issue:** Ubuntu Server ships with
> `/etc/ssh/sshd_config.d/50-cloud-init.conf` which overrides the main sshd_config.
> Simply editing sshd_config is not sufficient — the cloud-init file must be
> edited directly. See [Issue 2](#issue-2----ubuntu-ssh-not-binding-to-port-2222-due-to-cloud-init-override).

> **Ubuntu sshd.socket conflict:** Ubuntu also runs `sshd.socket` alongside
> `ssh.service` which can prevent the service from binding to the new port.
> See [Issue 3](#issue-3----ubuntu-sshd-socket-conflict-preventing-port-change).

```bash
# Stop and disable conflicting services on Ubuntu
sudo systemctl stop sshd.socket
sudo systemctl disable sshd.socket
sudo systemctl stop sshd-keygen.service
sudo systemctl disable sshd-keygen.service
sudo systemctl restart ssh
sudo ss -tlnp | grep ssh
```

![ubuntu-node SSH into rhel-node, sshd.socket disabled](screenshots/5-3%20ubuntu%20ssh%20to%20rhel%20and%20disable%20sshd.PNG)

### Configure fail2ban on Ubuntu Node

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
logpath = /var/log/auth.log
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status sshd
```

![fail2ban status sshd showing active monitoring](screenshots/5-4%20fail2ban.PNG)

---

## 4. Sudo Policy Configuration

Perform on both nodes.

### Configure Group-Based Sudo Policies

Always use `visudo` to prevent syntax errors:

```bash
sudo visudo
```

Add at the bottom:

```
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
```

![visudo group policies configured on rhel-node](screenshots/6-1%20visudo%201.PNG)
![visudo group policies configured on ubuntu-node](screenshots/6-2%20visudo%202.PNG)

---

## 5. Firewall Configuration with firewalld

`firewalld` was used on both nodes for a consistent toolset across distributions.

### Install and Enable firewalld

```bash
# RHEL (pre-installed)
sudo systemctl enable firewalld
sudo systemctl start firewalld

# Ubuntu
sudo apt install firewalld -y
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

### Configure Zones and Rules

```bash
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

![ubuntu-node firewall rules configured](screenshots/7%20ubuntu%20firewall.PNG)

### Block a Specific IP (Threat Response Simulation)

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" \
  source address="192.168.1.100" reject'
sudo firewall-cmd --reload
```

![rhel-node blocking specific IP with rich rule](screenshots/7-2%20block%20specific%20IP%20rhel%20node.PNG)

---

## 6. Systemd Service Management

Performed on the RHEL node as part of RHCSA exam preparation.

### Create a Custom Service

```bash
sudo mkdir -p /opt/myapp
echo 'import time; print("App running"); time.sleep(3600)' | \
  sudo tee /opt/myapp/app.py
sudo chown bob:bob /opt/myapp/app.py
sudo chmod 755 /opt/myapp/app.py
```

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

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
sudo systemctl status myapp.service
```

![myapp.service active and running on rhel-node](screenshots/8-1%20service%20on%20rhel%20node.PNG)

### Simulate a Failure and Recovery

> **Important:** Use `SIGKILL` not `systemctl kill` (which sends SIGTERM).
> SIGTERM is a graceful shutdown that systemd treats as a successful stop,
> so `Restart=on-failure` does not trigger. SIGKILL forces an unclean
> termination which correctly triggers the restart policy.

```bash
sudo systemctl kill -s SIGKILL myapp.service
sleep 5
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -n 20
```

Look for this sequence in the journal confirming crash and auto-recovery:

```
Main process exited, code=killed, status=9/KILL
myapp.service: Scheduled restart job
Started My Application Service
```

![SIGKILL failure and automatic restart recovery](screenshots/8-2%20failure%20recovery.PNG)
![journalctl showing crash and recovery sequence](screenshots/8-3%20journalctl.PNG)

---

## 7. Centralized Logging with rsyslog

The RHEL node acts as a centralized log server receiving forwarded events from
the Ubuntu node. This simulates enterprise log aggregation prior to SIEM ingestion.

### Configure RHEL Node as Log Server

```bash
sudo vim /etc/rsyslog.conf
```

Uncomment or add:

```
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

```bash
sudo vim /etc/rsyslog.d/remote.conf
```

```
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```

```bash
sudo mkdir -p /var/log/remote
sudo chmod 755 /var/log/remote
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --add-port=514/udp
sudo firewall-cmd --reload
sudo systemctl restart rsyslog
```

![rhel-node log server configuration and rsyslog status](screenshots/9-1%20rhel%20node%20log%20server.PNG)

### Configure Ubuntu Node as Log Client

```bash
sudo vim /etc/rsyslog.d/50-remote.conf
```

```
*.* @@192.168.56.10:514
```

```bash
sudo systemctl restart rsyslog
```

### Verify Log Forwarding

```bash
# On ubuntu-node
logger -t "lab-test" "Test log forwarding from ubuntu-node"

# On rhel-node
sudo ls /var/log/remote/ubuntu-node/
sudo tail -f /var/log/remote/ubuntu-node/lab-test.log
```

![Log forwarding verified — message from ubuntu-node appears on rhel-node](screenshots/9-2%20log%20forwarding.PNG)

> **Note:** Log forwarding worked on first attempt with no troubleshooting required.
> Port 514 firewall rules must be added as permanent rules — runtime-only rules
> are lost on firewall reload. See [Scenario 6](#scenario-6----rsyslog-forwarding-breaks-after-firewall-reload)
> for the break/fix troubleshooting scenario.

---

## 8. Monitoring with Prometheus & Grafana

A full metrics monitoring stack deployed on ubuntu-node with Node Exporter on
both nodes. Prometheus scrapes metrics every 15 seconds. Grafana provides
dashboards, alerting, and email notifications.

**Why Prometheus + Grafana over alternatives like SCOM or SolarWinds:**
Prometheus and Grafana are open source, widely used in cloud-native and
hybrid environments, and directly referenced in multiple sysadmin and cloud
administrator job descriptions. SCOM requires Windows Server infrastructure
and enterprise licensing. SolarWinds Orion is enterprise-licensed and
cost-prohibitive for a home lab. Prometheus + Grafana provides equivalent
monitoring capability with no licensing cost.

### Install Node Exporter on Both Nodes

Perform on both rhel-node and ubuntu-node:

```bash
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xzf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
```

> **Path note:** The binary is inside a subdirectory in the archive. Confirm
> the exact path before moving: `ls node_exporter-1.8.1.linux-amd64/`

```bash
sudo useradd -rs /bin/false node_exporter

sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

> **RHEL SELinux note:** On RHEL, any new binary placed in `/usr/local/bin/`
> must have `restorecon` run on it before starting as a service. SELinux will
> block execution with exit code 203/EXEC without this step.

```bash
# On rhel-node — required before starting the service
sudo restorecon -v /usr/local/bin/node_exporter
sudo systemctl reset-failed node_exporter
sudo systemctl start node_exporter
```

On rhel-node also open port 9100:

```bash
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
```

![ubuntu-node Node Exporter setup and service status](screenshots/10-1%20ubuntu%20node%20exporter%20setup.PNG)
![rhel-node Node Exporter SELinux issue and restorecon fix](screenshots/10-2%20rhel%20node%20exporter%20issue.PNG)

Verify metrics on both nodes:

```bash
curl http://192.168.56.10:9100/metrics | head -20
curl http://192.168.56.20:9100/metrics | head -20
```

![Metric data being served on both nodes](screenshots/10-3%20metric%20data.PNG)

### Install Prometheus on Ubuntu Node

> **Version note:** Prometheus v3.x removed the `consoles` and `console_libraries`
> directories from the release package. Skip the `sudo mv` command for those
> directories — they no longer exist in v3.x releases.

```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v3.11.3/prometheus-3.11.3.linux-amd64.tar.gz
tar -xzf prometheus-3.11.3.linux-amd64.tar.gz
sudo mv prometheus-3.11.3.linux-amd64/{prometheus,promtool} /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
# Note: do NOT run the consoles/console_libraries mv command — removed in v3.x
```

### Configure Prometheus

> **Always validate prometheus.yml before starting the service:**
> `promtool check config /etc/prometheus/prometheus.yml`
> This catches YAML errors immediately and shows the exact line number.
> Common mistake: `scrape-configs` (hyphen) instead of `scrape_configs` (underscore).

```bash
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets:
          - "192.168.56.10:9100"
          - "192.168.56.20:9100"
        labels:
          env: "lab"
EOF

promtool check config /etc/prometheus/prometheus.yml
```

```bash
sudo useradd -rs /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9090
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

> **RHEL SELinux note:** Run `sudo restorecon -v /usr/local/bin/prometheus`
> before starting if the service fails with exit code 203/EXEC.

![Prometheus service active running on ubuntu-node](screenshots/10-4%20prometheus%20running.PNG)

### Host Browser Access via SSH Tunnel

> **VirtualBox port forwarding did not work** in this environment despite correct
> configuration. The reliable solution is an SSH tunnel from the host machine.
> See [Issue 8](#issue-8----host-browser-cannot-reach-prometheus-or-grafana)
> for full details.

**Add SSH port forwarding rule to ubuntu-node's NAT adapter:**

1. Shut down ubuntu-node
2. Settings > Network > Adapter 2 (NAT) > Port Forwarding
3. Add rule: Name=SSH, Protocol=TCP, Host Port=2222, Guest Port=2222
4. Start ubuntu-node

**Add host machine's public key to ubuntu-node authorized_keys**, then run
this command on the host machine each session:

```powershell
ssh -p 2222 -L 9090:localhost:9090 -L 3000:localhost:3000 alice@127.0.0.1
```

Keep this PowerShell window open. Then access from host browser:

```
http://localhost:9090/targets
http://localhost:3000
```

![Prometheus targets page on host browser via SSH tunnel](screenshots/10-5%20prometheus%20host%20browser%20after%20issue.PNG)

### Install Grafana on Ubuntu Node

> **apt repository method failed** on Ubuntu 26.04 — the source list file
> format was not accepted. Used direct .deb download instead.
> See [Issue 9](#issue-9----grafana-apt-repository-method-failed-on-ubuntu-2604).

```bash
# Install dependencies first
sudo apt-get install -y adduser libfontconfig1 musl

# Download and install Grafana OSS directly
wget https://dl.grafana.com/oss/release/grafana_13.0.1_linux_amd64.deb
sudo dpkg -i grafana_13.0.1_linux_amd64.deb

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

Access at `http://localhost:3000` (admin/admin, change on first login).

![Grafana login page on host browser via SSH tunnel](screenshots/10-6%20grafana%20host%20browser.PNG)

### Connect Prometheus as Data Source

1. Connections > Data Sources > Add data source > Prometheus
2. URL: `http://localhost:9090`
3. Save & Test

### Import Node Exporter Dashboard

Dashboards > Import > ID: `1860` > Load > Select Prometheus > Import

![Node Exporter Full dashboard in Grafana showing both nodes](screenshots/10-7%20node%20exporter%20dashboard.PNG)

### Configure Email Alerts (SMTP)

```bash
sudo vim /etc/grafana/grafana.ini
```

```ini
[smtp]
enabled = true
host = smtp.gmail.com:587
user = your.gmail@gmail.com
password = your-16-char-app-password
from_address = your.gmail@gmail.com
from_name = Grafana Lab
startTLS_policy = MandatoryStartTLS
```

```bash
sudo systemctl restart grafana-server
```

> Gmail requires an App Password (not your regular password). Generate one at
> myaccount.google.com > Security > 2-Step Verification > App passwords.

![Grafana test email received confirming SMTP configuration](screenshots/10-8%20grafana%20test%20email.PNG)

### Configure Disk Usage Alert

In Grafana > Alerting > Alert Rules > New Alert Rule:

- Switch to **Code** mode and enter the PromQL query
- Set threshold: Input A is above 80
- Create a folder and evaluation group (evaluate every 1m, keep firing 5m)
- Add contact point with email
- Add custom annotation before saving

```
(node_filesystem_size_bytes{job="node_exporter",fstype!="tmpfs"} -
 node_filesystem_avail_bytes{job="node_exporter",fstype!="tmpfs"}) /
 node_filesystem_size_bytes{job="node_exporter",fstype!="tmpfs"} * 100 > 80
```

![Disk usage alert rule configured in Grafana](screenshots/10-9%20grafana%20disk%20usage%20alert.PNG)

### Simulate a Performance Incident

> **stress-ng failed on RHEL** due to insufficient available memory (only 256 MB
> free of 666 MB total after OS overhead). Used bash `yes` command instead for
> equivalent CPU load generation.

```bash
# On rhel-node — bash CPU load simulation
yes > /dev/null &
yes > /dev/null &
sleep 60
kill $(pgrep yes)
```

```bash
# On ubuntu-node — stress works fine
stress --cpu 2 --vm 1 --vm-bytes 512M --timeout 60s
```

Run one node at a time to see distinct per-node spikes in Grafana.

![rhel-node CPU spike in Grafana dashboard during stress test](screenshots/10-10%20rhel%20stress.PNG)
![ubuntu-node CPU spike in Grafana dashboard during stress test](screenshots/10-11%20ubuntu%20stress.PNG)

---

## 9. Container Management with Docker

Docker was installed on ubuntu-node to practice container lifecycle management
and demonstrate a containerized alternative to the manual monitoring stack deployment.

**Why Docker over alternatives:**
Docker is the most widely referenced container runtime in sysadmin and cloud
administrator job descriptions. Podman (RHEL's default) is the natural choice
on RHEL but Docker provides broader cross-platform familiarity and Docker Compose
is more commonly referenced in JDs for infrastructure roles.

### Install Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo systemctl enable docker
sudo systemctl start docker

# Add alice to docker group to avoid sudo on every command
sudo usermod -aG docker $USER
newgrp docker
```

### Core Docker Operations

```bash
docker run -d --name webserver -p 8080:80 nginx
docker ps
docker logs webserver
docker exec -it webserver bash
docker inspect webserver
docker stop webserver
docker rm webserver
```

![Docker core operations — run, ps, logs, exec, inspect](screenshots/11-1%20docker%20commands.PNG)
![Docker stop, rm, and additional commands](screenshots/11-2%20docker%20commands.PNG)

### Docker Compose — Monitoring Stack

> Before deploying, stop the systemd Prometheus and Grafana services to free
> ports 9090 and 3000:

```bash
sudo systemctl stop prometheus grafana-server
sudo systemctl disable prometheus grafana-server
```

```bash
mkdir -p ~/monitoring-stack && cd ~/monitoring-stack
```

Create `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets:
          - "192.168.56.10:9100"
          - "192.168.56.20:9100"
```

Create `docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    network_mode: host
    volumes:
      - /home/alice/monitoring-stack/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    network_mode: host
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

> **version: attribute removed** — Docker Compose v2 no longer requires the
> version field and will show a deprecation warning if included.

> **network_mode: host** is required so Prometheus containers can reach the
> node exporters on the internal lab network (192.168.56.x). Without it,
> containers are isolated to Docker's bridge network and cannot scrape metrics.

```bash
docker compose up -d
docker compose ps
```

> **Volume mount issue:** Despite correct absolute path configuration, the
> prometheus.yml volume mount did not work with `network_mode: host`. Config
> must be copied directly into the container after each startup.
> See [Issue 11](#issue-11----docker-compose-volume-mount-not-working-with-network_mode-host).

```bash
docker cp ~/monitoring-stack/prometheus.yml prometheus:/etc/prometheus/prometheus.yml
docker restart prometheus
```

![Docker Compose stack running with both containers up](screenshots/11-3%20docker%20compose.PNG)
![Docker Compose volume mount issue — prometheus.yml not picked up](screenshots/11-4%20docker%20compose%20issue.PNG)
![Prometheus running via Docker Compose with node_exporter targets](screenshots/11-5%20prometheus.PNG)
![Grafana running via Docker Compose](screenshots/11-6%20grafana.PNG)

---

## 10. Configuration Management with Ansible

Ansible installed on ubuntu-node as the control node, managing both lab nodes
over SSH. Playbooks automate tasks performed manually earlier in the lab.

**Why Ansible over alternatives like Chef or Puppet:**
Ansible is agentless — it uses SSH which is already configured. Chef and Puppet
require agents installed on managed nodes. Ansible's YAML playbook syntax is
also more readable and widely referenced in job descriptions for infrastructure
and cloud roles. For a two-node lab, Ansible's simplicity is the right fit.

### Install Ansible

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

### Configure Inventory and ansible.cfg

```bash
mkdir -p ~/ansible && cd ~/ansible
```

Create `inventory.ini`:

```ini
[rhel]
rhel-node ansible_host=192.168.56.10 ansible_user=alice ansible_port=2222

[ubuntu]
ubuntu-node ansible_host=192.168.56.20 ansible_user=alice ansible_port=2222

[lab:children]
rhel
ubuntu
```

> **Connection note:** Do not use `ansible_connection=local` for ubuntu-node
> even though it is the control node. Local connection causes privilege
> escalation timeouts. Use SSH for both nodes — ubuntu-node SSHs to itself
> which works correctly with key-based auth.

Create `ansible.cfg`:

```bash
vim ~/ansible/ansible.cfg
```

```ini
[defaults]
inventory = inventory.ini
remote_user = alice
host_key_checking = false

[ssh_connection]
pipelining = true
```

> **pipelining = true** eliminates sftp/scp transfer warnings that appear when
> Ansible tries to transfer module code to managed nodes.

Test connectivity:

```bash
ansible all -i inventory.ini -m ping
```

Both nodes should return `pong`.

> **NOPASSWD sudo required for Ansible:** Ansible's non-interactive sudo
> invocation requires passwordless sudo on both nodes. Add to `/etc/sudoers`
> directly via visudo on each node — sudoers.d drop-in files were unreliable
> on RHEL. See [Issue 13](#issue-13----ansible-privilege-escalation-fails-on-rhel).

```bash
sudo visudo
# Add at the bottom:
Defaults:alice !requiretty
alice ALL=(ALL) NOPASSWD: ALL
```

![Ansible inventory configured and ping test returning pong on both nodes](screenshots/12-1%20inventory.PNG)

### Playbook 1 — User and Group Management

```yaml
# playbooks/users.yml
---
- name: Configure users and groups across all lab nodes
  hosts: lab
  become: true

  vars:
    lab_groups:
      - sysadmins
      - developers
      - auditors

    lab_users:
      - name: alice
        groups: sysadmins
      - name: bob
        groups: developers
      - name: carol
        groups: auditors

  tasks:
    - name: Create lab groups
      group:
        name: "{{ item }}"
        state: present
      loop: "{{ lab_groups }}"

    - name: Create lab users with group assignment
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        shell: /bin/bash
        create_home: true
        state: present
      loop: "{{ lab_users }}"

    - name: Set password aging policy
      command: chage -M 90 -W 14 {{ item.name }}
      loop: "{{ lab_users }}"
      changed_when: false
```

```bash
ansible-playbook -i inventory.ini playbooks/users.yml
```

![Users playbook output showing ok=3 on both nodes](screenshots/12-2%20users%20playbooks.PNG)

### Playbook 2 — Package Installation

> **Deprecation note:** Use `ansible_facts['os_family']` syntax instead of
> `ansible_os_family` to avoid deprecation warnings in Ansible 2.20+.

```yaml
# playbooks/packages.yml
---
- name: Install required packages on all nodes
  hosts: lab
  become: true

  tasks:
    - name: Install packages on RHEL
      dnf:
        name:
          - git
          - curl
          - wget
          - tree
          - net-tools
          - firewalld
          - rsyslog
        state: present
      when: ansible_facts['os_family'] == "RedHat"

    - name: Install packages on Ubuntu
      apt:
        name:
          - git
          - curl
          - wget
          - tree
          - net-tools
          - firewalld
          - rsyslog
          - fail2ban
        state: present
        update_cache: true
      when: ansible_facts['os_family'] == "Debian"
```

```bash
ansible-playbook -i inventory.ini playbooks/packages.yml
```

> **skipped=1 is expected** — the RHEL task skips on ubuntu-node and the Ubuntu
> task skips on rhel-node. This is correct behavior from the `when` conditionals.

![Packages playbook output showing ok=2 skipped=1 on both nodes](screenshots/12-3%20packages%20playbooks.PNG)

### Playbook 3 — SSH Hardening Enforcement

```yaml
# playbooks/ssh_hardening.yml
---
- name: Enforce SSH hardening across all lab nodes
  hosts: lab
  become: true

  handlers:
    - name: Restart SSH service (RHEL)
      systemd:
        name: sshd
        state: restarted
      when: ansible_facts['os_family'] == "RedHat"

    - name: Restart SSH service (Ubuntu)
      systemd:
        name: ssh
        state: restarted
      when: ansible_facts['os_family'] == "Debian"

  tasks:
    - name: Deploy hardened sshd_config
      copy:
        dest: /etc/ssh/sshd_config
        content: |
          PermitRootLogin no
          PasswordAuthentication no
          MaxAuthTries 3
          ClientAliveInterval 300
          ClientAliveCountMax 0
          Port 2222
          AllowUsers alice
        owner: root
        group: root
        mode: "0600"
      notify:
        - Restart SSH service (RHEL)
        - Restart SSH service (Ubuntu)

    - name: Validate sshd config syntax
      command: sshd -t
      changed_when: false
```

```bash
ansible-playbook -i inventory.ini playbooks/ssh_hardening.yml
```

![SSH hardening playbook output on both nodes](screenshots/12-4%20ssh%20hardening%20playbooks.PNG)

### Playbook 4 — Service Enforcement

```yaml
# playbooks/services.yml
---
- name: Enforce service states across all lab nodes
  hosts: lab
  become: true

  vars:
    common_services:
      - rsyslog
      - firewalld

    rhel_services:
      - sshd

    ubuntu_services:
      - ssh
      - fail2ban

  tasks:
    - name: Ensure common services are running and enabled
      systemd:
        name: "{{ item }}"
        state: started
        enabled: true
      loop: "{{ common_services }}"

    - name: Ensure RHEL-specific services are running
      systemd:
        name: "{{ item }}"
        state: started
        enabled: true
      loop: "{{ rhel_services }}"
      when: ansible_facts['os_family'] == "RedHat"

    - name: Ensure Ubuntu-specific services are running
      systemd:
        name: "{{ item }}"
        state: started
        enabled: true
      loop: "{{ ubuntu_services }}"
      when: ansible_facts['os_family'] == "Debian"
```

```bash
ansible-playbook -i inventory.ini playbooks/services.yml
```

![Services playbook output showing ok=3 skipped=1 on both nodes](screenshots/12-5%20services%20playbooks.PNG)

---

## 11. Bash Automation Scripts

Scripts developed on ubuntu-node following enterprise operational runbook
practices with timestamped logging and syslog integration via `logger`.

> **Prerequisites before running scripts:**
> - Create log file: `sudo touch /var/log/health_check.log && sudo chown alice:alice /var/log/health_check.log`
> - Create backup dirs: `mkdir -p ~/Documents ~/projects ~/.config`
> - Create remote backup dir on rhel-node: `mkdir -p ~/remote_backups`

### Backup Script

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

# Only include directories that exist
SPECIFIC_DIRS=()
for dir in "$HOME/Documents" "$HOME/projects" "$HOME/.config" \
           "$HOME/ansible" "$HOME/monitoring-stack" "/etc"; do
  if [ -d "$dir" ]; then
    SPECIFIC_DIRS+=("$dir")
  fi
done

SOURCE_DIRS=("${SPECIFIC_DIRS[@]}")

mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/$FILENAME" "${SOURCE_DIRS[@]}" 2>/dev/null
logger -t "$LOG_TAG" "Backup created: $BACKUP_DIR/$FILENAME"

ssh -p 2222 "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p ${REMOTE_BACKUP_DIR}"
scp -P 2222 "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BACKUP_DIR}/*.tar.gz" \
  "$BACKUP_DIR/" 2>/dev/null
logger -t "$LOG_TAG" "Remote backup pulled from ${REMOTE_HOST}"

find "$BACKUP_DIR" -name "*.tar.gz" -type f -mtime +$RETENTION_DAYS -exec rm {} \;
logger -t "$LOG_TAG" "Old backups older than $RETENTION_DAYS days removed"

echo "Backup complete: $FILENAME"
```

![Backup script running cleanly with no errors](screenshots/13-1%20backup.PNG)

### System Health Check Script

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
    cpu=$(ssh -p 2222 "${REMOTE_USER}@${REMOTE_HOST}" \
      "top -bn1 | grep 'Cpu(s)' | awk '{print \$2}' | cut -d. -f1")
    mem=$(ssh -p 2222 "${REMOTE_USER}@${REMOTE_HOST}" \
      "free | awk '/Mem:/ {printf \"%.0f\", \$3/\$2 * 100}'")
    disk=$(ssh -p 2222 "${REMOTE_USER}@${REMOTE_HOST}" \
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

> **Prerequisite:** Create and chown the log file before running:
> `sudo touch /var/log/health_check.log && sudo chown alice:alice /var/log/health_check.log`

![Health check script output and log file showing readings for both nodes](screenshots/13-2%20health%20check.PNG)

### Log Rotation Script

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

find "$LOG_DIR" -name "*.log" -mtime +$RETENTION_DAYS 2>/dev/null | \
  while read logfile; do
    BASENAME=$(basename "$logfile")
    gzip -c "$logfile" > "$ARCHIVE_DIR/${BASENAME}_${DATE}.gz" 2>/dev/null
    echo "Archived: $logfile"
    logger -t "log-rotate" "Archived: $logfile"
  done

find "$ARCHIVE_DIR" -name "*.gz" -mtime +30 -exec rm {} \;
logger -t "log-rotate" "Removed archives older than 30 days"
```

> **2>/dev/null on both find and gzip** suppresses permission denied errors
> for system log files that alice cannot read. This is expected behavior —
> in production log rotation runs as root via cron for full access.

![Log rotation script archiving accessible logs](screenshots/13-3%20log%20rotation.PNG)

### Make Executable and Schedule with Cron

```bash
chmod +x ~/backup.sh ~/health_check.sh ~/log_rotate.sh
crontab -e
```

```
# Backup daily at 1:00 AM
0 1 * * * /home/alice/backup.sh

# Health check every 15 minutes
*/15 * * * * /home/alice/health_check.sh

# Log rotation weekly on Sunday at 2:00 AM
0 2 * * 0 /home/alice/log_rotate.sh
```

![crontab -l showing all three scheduled jobs](screenshots/13-4%20crontab%20-l.PNG)

---

## 12. Troubleshooting Scenarios

Real failure scenarios encountered and reproduced, simulating production
incident response. Scenarios marked **[RHEL]** were reproduced on rhel-node
as part of RHCSA exam preparation.

---

### Scenario 1 — Service Fails to Start [RHEL]

**Break it:**

```bash
sudo chmod 000 /opt/myapp/app.py
sudo systemctl restart myapp.service
```

**Diagnose:**

```bash
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -n 50
sudo ausearch -m avc -ts recent
```

**Fix it:**

```bash
sudo chmod 755 /opt/myapp/app.py
sudo chown bob:bob /opt/myapp/app.py
sudo restorecon -v /opt/myapp/app.py
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

![Scenario 1 — service failure diagnosis and fix](screenshots/14-1%20scenario%201.PNG)

---

### Scenario 2 — SSH Login Denied After Config Change

**Break it:**

```bash
sudo bash -c 'echo "InvalidOption yes" >> /etc/ssh/sshd_config'
sudo systemctl restart sshd
```

**Diagnose:**

```bash
sudo systemctl status sshd
sudo sshd -t
```

**Fix it:**

```bash
sudo sed -i '/InvalidOption/d' /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl restart sshd
```

![Scenario 2 — SSH config error detected with sshd -t and resolved](screenshots/14-2%20scenario%202.PNG)

---

### Scenario 3 — Disk Space Exhaustion

**Simulate it:**

```bash
dd if=/dev/zero of=/tmp/bigfile bs=1M count=5000
```

**Diagnose:**

```bash
df -h
du -sh /* 2>/dev/null | sort -rh | head -20
du -sh /tmp/* | sort -rh | head -10
```

![Scenario 3 — disk exhaustion and df -h showing full partition](screenshots/14-3%20scenario%203%20p1.PNG)

**Fix it:**

```bash
rm /tmp/bigfile
sudo journalctl --vacuum-time=7d
df -h
```

![Scenario 3 — disk space recovered after cleanup](screenshots/14-4%20scenario%203%20p2.PNG)

---

### Scenario 4 — fail2ban Blocking Legitimate User

> **Note:** This scenario was not fully demonstrable due to SSH hardening.
> See [Issue 6](#issue-6----fail2ban-scenario-not-demonstrable-with-hardened-ssh).

---

### Scenario 5 — SELinux Blocking Service [RHEL]

**Break it:**

```bash
sudo mkdir -p /opt/myapp/logs
sudo chcon -t httpd_log_t /opt/myapp/logs
```

**Diagnose:**

```bash
sudo ausearch -m avc -ts recent
sudo cat /var/log/audit/audit.log | grep denied
ls -Z /opt/myapp/
```

**Fix it:**

```bash
sudo semanage fcontext -a -t var_log_t "/opt/myapp/logs(/.*)?"
sudo restorecon -Rv /opt/myapp/logs/
ls -Z /opt/myapp/logs/
```

![Scenario 5 — SELinux context fix with semanage and restorecon](screenshots/14-5%20scenario%205.PNG)

---

### Scenario 6 — rsyslog Forwarding Breaks After Firewall Reload

**Break it:**

```bash
sudo firewall-cmd --remove-port=514/tcp
sudo firewall-cmd --remove-port=514/udp
sudo firewall-cmd --reload
```

**Diagnose:**

```bash
sudo firewall-cmd --list-all | grep 514
nc -zv 192.168.56.10 514
sudo systemctl status rsyslog
```

![Scenario 6 — port 514 missing after firewall reload](screenshots/14-6%20scenario%206%20p1.PNG)

**Fix it:**

```bash
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --add-port=514/udp
sudo firewall-cmd --reload
sudo systemctl restart rsyslog
logger -t "lab-test" "Forwarding restored"
sudo tail -f /var/log/remote/ubuntu-node/lab-test.log
```

![Scenario 6 — forwarding restored and verified](screenshots/14-7%20scenario%206%20p2.PNG)

---

### Scenario 7 — Docker Container Restarting in a Loop

**Break it:**

```bash
docker exec -it prometheus sh -c 'echo "bad_config: true" >> /etc/prometheus/prometheus.yml'
docker restart prometheus
```

**Diagnose:**

```bash
docker ps -a
docker logs --tail 50 prometheus
docker inspect prometheus | grep "ExitCode"
```

**Fix it:**

```bash
docker cp ~/monitoring-stack/prometheus.yml prometheus:/etc/prometheus/prometheus.yml
docker restart prometheus
docker compose ps
```

![Scenario 7 — container crash diagnosed and fixed with docker cp](screenshots/14-8%20scenario%207.PNG)

---

### Scenario 8 — Ansible Package Name Mismatch Between Distributions

**Simulate it:**

```bash
cat > ~/ansible/playbooks/test_mismatch.yml << 'EOF'
---
- name: Test package mismatch
  hosts: lab
  become: true
  tasks:
    - name: Install httpd-tools on all nodes
      package:
        name: httpd-tools
        state: present
EOF

ansible-playbook -i inventory.ini playbooks/test_mismatch.yml -v
```

![Scenario 8 — playbook failing on Ubuntu due to wrong package name](screenshots/14-9%20scenario%208%20p1.PNG)

**Fix it:**

```bash
cat > ~/ansible/playbooks/test_mismatch_fixed.yml << 'EOF'
---
- name: Install web utilities correctly per distribution
  hosts: lab
  become: true
  tasks:
    - name: Install on RHEL
      dnf:
        name: httpd-tools
        state: present
      when: ansible_facts['os_family'] == "RedHat"

    - name: Install on Ubuntu
      apt:
        name: apache2-utils
        state: present
      when: ansible_facts['os_family'] == "Debian"
EOF

ansible-playbook -i inventory.ini playbooks/test_mismatch_fixed.yml
```

![Scenario 8 — fixed with os_family conditionals running successfully](screenshots/14-10%20scenario%208%20p2%20after%20fix.PNG)

---

## 13. RHCSA Exam Practice

This lab served as the primary preparation environment for the RHCSA exam
(EX200), passed in April 2026. All tasks were practiced on the RHEL node.

### Task 1 — Create Users with Supplementary Groups

```bash
sudo groupadd -g 2001 developers
sudo groupadd -g 2002 sysadmins
sudo useradd -u 1050 -g developers -G sysadmins -m -s /bin/bash john
id john
```

### Task 2 — Configure a Cron Job for a Specific User

```bash
sudo crontab -u alice -e
```
```
30 3 * * * /home/alice/backup.sh
```
```bash
sudo crontab -u alice -l
```

### Task 3 — Set File Permissions and ACLs

```bash
sudo mkdir -p /data/shared
sudo chown root:developers /data/shared
sudo chmod 770 /data/shared
sudo dnf install acl -y
sudo setfacl -m g:auditors:r-x /data/shared
getfacl /data/shared
```

### Task 4 — Configure a Systemd Service to Start on Boot

```bash
sudo systemctl enable myapp.service
sudo systemctl reboot
sudo systemctl status myapp.service
```

### Task 5 — Diagnose and Fix a Failed Service

```bash
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -n 30
sudo ausearch -m avc -ts recent
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

### Task 6 — Manage Password Policies

```bash
sudo chage -M 60 -W 7 bob
sudo chage -l bob
```

### Task 7 — Configure SELinux Modes and Contexts

```bash
getenforce
sestatus
sudo setenforce 0
sudo setenforce 1
sudo vim /etc/selinux/config   # Set: SELINUX=enforcing
sudo restorecon -Rv /opt/myapp/
```

### Task 8 — Configure a Network Interface with nmcli

```bash
sudo nmcli con show
sudo nmcli con mod "enp0s3" \
  ipv4.addresses 192.168.56.10/24 \
  ipv4.method manual
sudo nmcli con up "enp0s3"
ip a
```

---

## 14. Issues & Lessons Learned

All issues encountered during the lab, documented with root cause and resolution.

---

### Issue 1 — RHEL NAT Adapter Requires Manual nmcli Connection Creation

**Lab section:** VM Setup
**Symptom:** Adding NAT adapter in VirtualBox and starting VM did not
automatically configure enp0s8 with a DHCP address on RHEL.

**Root Cause:** RHEL's NetworkManager does not auto-create a connection
profile for newly added network adapters, unlike Ubuntu which handles this
automatically via cloud-init.

**Resolution:**

```bash
sudo nmcli con add type ethernet ifname enp0s8 con-name "NAT" ipv4.method auto
sudo nmcli con up "NAT"
```

**Lesson:** Run `nmcli con up "NAT"` any time internet access is needed on
rhel-node throughout the lab.

---

### Issue 2 — Ubuntu SSH Not Binding to Port 2222 Due to cloud-init Override

**Lab section:** Part 3 — SSH Hardening
**Symptom:** After editing sshd_config and restarting SSH, the service still
listened on port 22. `ss -tlnp | grep 2222` showed nothing.

**Root Cause:** Ubuntu Server ships with `/etc/ssh/sshd_config.d/50-cloud-init.conf`
containing `PasswordAuthentication yes` which overrides the main sshd_config.
Any settings in this file take precedence over the main config.

**Resolution:** Edit `/etc/ssh/sshd_config.d/50-cloud-init.conf` directly:

```
Port 2222
PasswordAuthentication no
```

---

### Issue 3 — Ubuntu sshd.socket Conflict Preventing Port Change

**Lab section:** Part 3 — SSH Hardening
**Symptom:** Even after editing 50-cloud-init.conf correctly, SSH still
bound to port 22. `ss -tlnp | grep ssh` showed port 22 in use.

**Root Cause:** Ubuntu runs both `ssh.service` and `sshd.socket` simultaneously.
The socket unit was intercepting connections on port 22 independently of the
service configuration.

**Resolution:**

```bash
sudo systemctl stop sshd.socket
sudo systemctl disable sshd.socket
sudo systemctl stop sshd-keygen.service
sudo systemctl disable sshd-keygen.service
sudo systemctl restart ssh
sudo ss -tlnp | grep ssh  # Now shows port 2222
```

---

### Issue 4 — SIGTERM Does Not Trigger Restart=on-failure

**Lab section:** Part 6 — Systemd Service Management
**Symptom:** `systemctl kill myapp.service` showed the service stopping but
not restarting automatically.

**Root Cause:** `systemctl kill` sends SIGTERM by default which is a graceful
shutdown. When a process exits cleanly with code 0, systemd considers it a
successful stop and does not trigger `Restart=on-failure`.

**Resolution:** Use SIGKILL to force an unclean termination:

```bash
sudo systemctl kill -s SIGKILL myapp.service
```

**Lesson:** `Restart=on-failure` only triggers on unclean exits. Use SIGKILL
to reliably simulate crashes in a lab environment.

---

### Issue 5 — Node Exporter Fails With Exit Code 203/EXEC on RHEL

**Lab section:** Part 8 — Prometheus & Grafana
**Symptom:** `systemctl start node_exporter` fails immediately with
`status=203/EXEC`. Service restart counter hits limit and stops.

**Root Cause:** Two issues:
1. Binary extracted to wrong subdirectory path
2. SELinux blocking execution of newly placed binary

**Resolution:**

```bash
sudo ausearch -m avc -ts recent | grep node_exporter
sudo restorecon -v /usr/local/bin/node_exporter
sudo systemctl reset-failed node_exporter
sudo systemctl start node_exporter
```

**Lesson:** On RHEL, always run `restorecon` on any new binary placed in
`/usr/local/bin/` before starting it as a service. SELinux context issues
always surface as `203/EXEC` in systemd.

---

### Issue 6 — fail2ban Scenario Not Demonstrable With Hardened SSH

**Lab section:** Part 12 — Troubleshooting Scenario 4
**Symptom:** Unable to simulate SSH brute force attempts to trigger fail2ban.

**Root Cause:** SSH hardening (PasswordAuthentication no, key-only auth)
prevents the attack vector fail2ban is designed to protect against.
Invalid users receive immediate permission denied before fail2ban counts
the attempt. Valid users authenticate via key without a password prompt.

**Conclusion:** This is the desired outcome. A properly hardened SSH
configuration makes brute force attacks ineffective before fail2ban
needs to intervene. fail2ban serves as a secondary defense for environments
where password authentication must remain enabled.

---

### Issue 7 — stress-ng Fails on RHEL Due to Insufficient Memory

**Lab section:** Part 8 — Performance Incident Simulation
**Symptom:** `stress-ng --cpu 2 --vm 1 --vm-bytes 512M` fails with
`stressor failed exit status=2`. Reducing to 256M also fails — only
256 MB of 666 MB total was available after OS overhead.

**Resolution:** Used bash `yes` command for equivalent CPU load:

```bash
yes > /dev/null &
yes > /dev/null &
sleep 60
kill $(pgrep yes)
```

---

### Issue 8 — Host Browser Cannot Reach Prometheus or Grafana

**Lab section:** Part 8 — Prometheus & Grafana
**Symptom:** VirtualBox NAT port forwarding configured correctly
(netstat confirmed VirtualBox listening) but browser connections
timed out or were refused.

**Root Cause:** VirtualBox port forwarding did not function reliably
in this network configuration despite correct setup. Exact cause
not determined — possibly Windows Defender interaction or VirtualBox
version-specific behavior.

**Resolution:** SSH tunneling from host machine:

```powershell
ssh -p 2222 -L 9090:localhost:9090 -L 3000:localhost:3000 alice@127.0.0.1
```

Keep the SSH session open and access `http://localhost:9090` and
`http://localhost:3000` from the host browser.

**Lesson:** SSH tunneling is a realistic enterprise skill — it is
commonly used to securely access internal services without exposing
ports directly. This is arguably a better solution than port forwarding.

---

### Issue 9 — Grafana apt Repository Method Failed on Ubuntu 26.04

**Lab section:** Part 8 — Grafana Install
**Symptom:** Adding Grafana apt repository source list produced
"Type 'dev' is not known" and "Malformed entry" errors blocking
all apt commands.

**Root Cause:** The pipe-based echo command for writing the source
list file introduced encoding or whitespace issues incompatible with
the apt source list format on Ubuntu 26.04.

**Resolution:** Direct .deb package download and installation:

```bash
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/oss/release/grafana_13.0.1_linux_amd64.deb
sudo dpkg -i grafana_13.0.1_linux_amd64.deb
```

---

### Issue 10 — Grafana Database Locked on First Container Startup

**Lab section:** Part 9 — Docker Compose
**Symptom:** Grafana container shows Up status but port 3000 not
binding. Logs show "Database locked, sleeping then retrying".

**Root Cause:** Grafana volume initialized incorrectly on first startup.

**Resolution:**

```bash
docker compose down
docker volume rm monitoring-stack_grafana_data
docker compose up -d
```

---

### Issue 11 — Docker Compose Volume Mount Not Working With network_mode: host

**Lab section:** Part 9 — Docker Compose
**Symptom:** `docker exec prometheus cat /etc/prometheus/prometheus.yml`
shows default config despite volume mount in docker-compose.yml.
Tried relative path, absolute path, and `:ro` flag — all failed.

**Root Cause:** `network_mode: host` combined with volume mounts behaved
inconsistently in this Docker version and configuration.

**Resolution:** Copy config directly into container after each startup:

```bash
docker cp ~/monitoring-stack/prometheus.yml prometheus:/etc/prometheus/prometheus.yml
docker restart prometheus
```

**Lesson:** In production, config files for containers are managed via
configuration management tools like Ansible rather than relying on volume
mounts. This is a realistic production pattern.

---

### Issue 12 — Ansible ubuntu-node Privilege Escalation Timeout

**Lab section:** Part 10 — Ansible
**Symptom:** `ansible_connection=local` on ubuntu-node caused
"Timeout (12s) waiting for privilege escalation prompt".

**Root Cause:** Local connection mode handles privilege escalation
differently from SSH and times out when waiting for sudo prompts.

**Resolution:** Remove `ansible_connection=local` from inventory —
ubuntu-node uses SSH to connect to itself, which requires its own
public key in authorized_keys:

```bash
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

---

### Issue 13 — Ansible Privilege Escalation Fails on RHEL

**Lab section:** Part 10 — Ansible
**Symptom:** "Missing sudo password" / "Escalation requires password"
on rhel-node despite NOPASSWD in /etc/sudoers.d/alice with correct
0440 permissions and `sudo -n whoami` returning root.

**Root Cause:** RHEL's sudoers.d drop-in file not honored for Ansible's
non-interactive sudo invocation. Possibly related to SELinux context on
the sudoers.d file or RHEL's stricter sudoers.d handling.

**Resolution:** Add NOPASSWD entry directly to /etc/sudoers via visudo:

```bash
sudo visudo
# Add:
Defaults:alice !requiretty
alice ALL=(ALL) NOPASSWD: ALL
```

**Lesson:** When sudoers.d drop-in files cause issues on RHEL, adding
entries directly to /etc/sudoers via visudo is the reliable fallback.

---

### Issue 14 — Ansible Deprecation Warning for ansible_os_family

**Lab section:** Part 10 — Ansible
**Symptom:** DEPRECATION WARNING about INJECT_FACTS_AS_VARS when using
`ansible_os_family` in when conditionals.

**Resolution:** Update all playbooks to use new syntax:

```yaml
# Old (deprecated)
when: ansible_os_family == "RedHat"

# New (correct)
when: ansible_facts['os_family'] == "RedHat"
```

---

### Issue 15 — Ansible sftp/scp Transfer Warnings

**Lab section:** Part 10 — Ansible
**Symptom:** `[WARNING]: sftp transfer mechanism failed` and
`[WARNING]: scp transfer mechanism failed` on every playbook run.

**Root Cause:** Ansible tries SFTP then SCP before falling back to
SSH pipelining. Both SFTP and SCP were unavailable but tasks still
completed via the pipelining fallback.

**Resolution:** Add `pipelining = true` to ansible.cfg:

```ini
[ssh_connection]
pipelining = true
```

---

## Tools Used

| Tool               | Purpose                                                          |
|--------------------|------------------------------------------------------------------|
| journalctl         | Service and system log analysis                                  |
| systemctl          | Service management and desired state enforcement                 |
| ss                 | Active connection and socket inspection                          |
| top / htop         | Real-time process and CPU/memory monitoring                      |
| df / du            | Disk usage analysis                                              |
| strace             | System call tracing for permission and I/O debugging             |
| firewall-cmd       | Firewall rule management (firewalld)                             |
| fail2ban-client    | Intrusion prevention status and ban management                   |
| chage              | Password aging policy enforcement                                |
| visudo             | Safe sudoers policy editing                                      |
| ausearch           | SELinux audit log analysis                                       |
| semanage           | SELinux policy management (ports, file contexts)                 |
| restorecon         | Restore default SELinux file contexts                            |
| nmcli              | Network interface configuration (RHEL)                           |
| rsyslog            | Centralized log collection and forwarding                        |
| Prometheus         | Metrics collection and time-series storage                       |
| Node Exporter      | System-level metrics exposure for Prometheus                     |
| Grafana            | Metrics visualization, dashboards, and alerting                  |
| promtool           | Prometheus config validation                                     |
| Docker / Compose   | Container lifecycle management and multi-service deployment      |
| Ansible            | Agentless configuration management and playbook automation       |
| stress / yes       | CPU and memory load generation for monitoring simulation         |
| logger             | Write entries to syslog from shell scripts                       |
| SSH tunneling      | Secure browser access to internal services from host machine     |

---

[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:kennymiranda000@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/kenneth-miranda-xyz)
