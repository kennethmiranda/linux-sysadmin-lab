# Linux Systems Administration & Automation Lab
 
A hands-on, multi-node Linux lab built on VirtualBox simulating enterprise server operations across RHEL and Ubuntu environments. This lab covers user and group management, SSH hardening, sudo policy enforcement, firewall configuration, systemd service management, centralized logging, infrastructure monitoring, container management, configuration management automation, and real-world troubleshooting — serving as the primary preparation environment for the Red Hat Certified System Administrator (RHCSA) exam, passed in April 2026.
 
---
 
## Lab Environment
 
| Component       | Node 1 — RHEL                                                                   | Node 2 — Ubuntu                                                                      |
|-----------------|---------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| OS              | RHEL 9.4                                                                        | Ubuntu 24.04.2 LTS                                                                   |
| Hostname        | rhel-node                                                                       | ubuntu-node                                                                          |
| Virtualization  | Oracle VirtualBox                                                               | Oracle VirtualBox                                                                    |
| Host OS         | Windows 11                                                                      | Windows 11                                                                           |
| Resources       | 2 vCPUs, 2 GB RAM, 25 GB                                                        | 2 vCPUs, 2 GB RAM, 25 GB                                                             |
| Primary Role    | RHCSA practice, firewalld, systemd, SELinux, user mgmt, centralized log server  | Bash automation, monitoring stack, Docker, Ansible control node, log client          |
 
Both VMs run on an internal VirtualBox network, enabling SSH between nodes and simulating a basic multi-server environment.
 
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
---
 
## 1. VM Setup
 
### VirtualBox Network Configuration
 
Both VMs are attached to the same Internal Network in VirtualBox, enabling direct communication between nodes:
 
- VM1 (rhel-node): Internal Network `labnet`, static IP `192.168.56.10`
- VM2 (ubuntu-node): Internal Network `labnet`, static IP `192.168.56.20`
### RHEL Node Setup
 
Download RHEL 9 ISO via Red Hat's free Developer Program at [developers.redhat.com](https://developers.redhat.com).
 
Create a new VM in VirtualBox:
 
- Name: `rhel-node`
- Type: Linux / Red Hat (64-bit)
- RAM: 2048 MB | CPU: 2 cores | Disk: 25 GB (dynamically allocated)
After installation, register with Red Hat and update:
 
```bash
sudo subscription-manager register --username <username> --auto-attach
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
 
Performed on both nodes to practice across distributions. RHEL uses `useradd`/`groupadd` identically to Ubuntu; differences in defaults (e.g., home directory creation) are noted where relevant.
 
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
 
> On RHEL, `useradd` does not create a home directory by default unless `-m` is explicitly passed — this is a common RHCSA exam gotcha.
 
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
 
SSH was hardened on both nodes. Cross-node SSH (rhel-node ↔ ubuntu-node) was also configured to simulate a real multi-server environment where admins move between systems using key-based authentication.
 
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
 
> On RHEL with SELinux enforcing, changing the SSH port requires updating the SELinux port policy before the service will bind to the new port:
 
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
# sysadmins — full sudo access
%sysadmins ALL=(ALL:ALL) ALL
 
# developers — can restart application services only
%developers ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp.service
 
# auditors — read-only commands only
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
 
`firewalld` is the default firewall manager on RHEL and was also installed on the Ubuntu node to practice a consistent toolset across both distributions.
 
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
 
Custom services were created and managed on the RHEL node as part of RHCSA exam preparation, where systemd service configuration is a core tested objective.
 
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
 
## 7. Centralized Logging with rsyslog
 
The RHEL node acts as a centralized log server, receiving forwarded syslog events from the Ubuntu node. This simulates a core enterprise logging pattern where system events are aggregated at a central collection point before being forwarded to a SIEM or log management platform.
 
### Configure RHEL Node as Log Server
 
Enable TCP and UDP reception in `/etc/rsyslog.conf` by uncommenting the following lines:
 
```bash
sudo vim /etc/rsyslog.conf
```
 
```
# Provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")
 
# Provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")
```
 
Create a template to store incoming logs organized by source hostname:
 
```bash
sudo vim /etc/rsyslog.d/remote.conf
```
 
```
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```
 
Create the remote log directory and set permissions:
 
```bash
sudo mkdir -p /var/log/remote
sudo chown syslog:adm /var/log/remote 2>/dev/null || \
  sudo chown root:root /var/log/remote
sudo chmod 755 /var/log/remote
```
 
Open port 514 in firewalld and restart rsyslog:
 
```bash
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --add-port=514/udp
sudo firewall-cmd --reload
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```
 
### Configure Ubuntu Node as Log Client
 
```bash
sudo vim /etc/rsyslog.d/50-remote.conf
```
 
```
# Forward all logs to rhel-node over TCP
*.* @@192.168.56.10:514
```
 
```bash
sudo systemctl restart rsyslog
```
 
### Verify Log Forwarding
 
Generate a test log event on the Ubuntu node:
 
```bash
logger -t "lab-test" "Test log forwarding from ubuntu-node"
```
 
On the RHEL node, confirm the event arrived:
 
```bash
sudo ls /var/log/remote/ubuntu-node/
sudo tail -f /var/log/remote/ubuntu-node/lab-test.log
```
 
### Troubleshooting Scenario — Forwarding Silently Breaks After Firewall Reload
 
**Symptom:** Logs from ubuntu-node stop arriving at rhel-node after a `firewall-cmd --reload`. No error appears on the Ubuntu node.
 
**Diagnosis:**
 
```bash
# On rhel-node — confirm port 514 is no longer open
sudo firewall-cmd --list-all | grep 514
 
# On ubuntu-node — test TCP connectivity to rhel-node on port 514
nc -zv 192.168.56.10 514
 
# Check rsyslog is still running on both nodes
sudo systemctl status rsyslog
```
 
**Root Cause:** A `firewall-cmd --reload` without `--permanent` lost the runtime rule for port 514. The permanent rule was never set correctly, so the port closed on reload.
 
**Resolution:**
 
```bash
# On rhel-node — re-add as permanent rule
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --add-port=514/udp
sudo firewall-cmd --reload
 
# Verify
sudo firewall-cmd --list-all | grep 514
 
# Restart rsyslog on both nodes
sudo systemctl restart rsyslog
 
# Re-test from ubuntu-node
logger -t "lab-test" "Forwarding restored"
sudo tail -f /var/log/remote/ubuntu-node/lab-test.log
```
 
---
 
## 8. Monitoring with Prometheus & Grafana
 
A full metrics monitoring stack was deployed on the Ubuntu node using Prometheus for metrics collection and Grafana for visualization and alerting. Node Exporter runs on both nodes, exposing system-level metrics (CPU, memory, disk, network) that Prometheus scrapes on a scheduled interval.
 
### Install Node Exporter on Both Nodes
 
Node Exporter exposes hardware and OS metrics for Prometheus to collect.
 
**On ubuntu-node:**
 
```bash
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xzf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
```
 
Create a dedicated system user and systemd service:
 
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
sudo systemctl status node_exporter
```
 
**On rhel-node** (repeat the same steps, then open the port in firewalld):
 
```bash
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
```
 
Verify Node Exporter is serving metrics on both nodes:
 
```bash
curl http://192.168.56.10:9100/metrics | head -20   # rhel-node
curl http://192.168.56.20:9100/metrics | head -20   # ubuntu-node
```
 
### Install Prometheus on Ubuntu Node
 
```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
tar -xzf prometheus-2.52.0.linux-amd64.tar.gz
sudo mv prometheus-2.52.0.linux-amd64/{prometheus,promtool} /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo mv prometheus-2.52.0.linux-amd64/{consoles,console_libraries} /etc/prometheus/
```
 
### Configure Prometheus to Scrape Both Nodes
 
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
          - "192.168.56.10:9100"   # rhel-node
          - "192.168.56.20:9100"   # ubuntu-node
        labels:
          env: "lab"
EOF
```
 
Create a system user and systemd service for Prometheus:
 
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
 
Verify Prometheus is running and can reach both targets by opening the web UI from your host browser:
 
```
http://192.168.56.20:9090/targets
```
 
Both rhel-node and ubuntu-node should show **State: UP**.
 
### Install Grafana on Ubuntu Node
 
```bash
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install grafana -y
 
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```
 
Access Grafana at `http://192.168.56.20:3000` (default credentials: `admin` / `admin`).
 
### Connect Prometheus as a Data Source
 
In the Grafana UI:
 
```
Configuration > Data Sources > Add data source > Prometheus
URL: http://localhost:9090
Click: Save & Test
```
 
### Build a Node Monitoring Dashboard
 
Import the community Node Exporter Full dashboard (ID `1860`) for an immediate production-quality view:
 
```
Dashboards > Import > Enter ID: 1860 > Load > Select Prometheus data source > Import
```
 
This dashboard displays per-node CPU usage, memory utilization, disk I/O, network throughput, filesystem capacity, and system load average — all updated at the configured 15-second scrape interval.
 
### Configure a Disk Usage Alert
 
In Grafana, create an alert rule that fires when disk usage on either node exceeds 80%:
 
```
Alerting > Alert Rules > New Alert Rule
 
Query:
  (node_filesystem_size_bytes{job="node_exporter",fstype!="tmpfs"} -
   node_filesystem_avail_bytes{job="node_exporter",fstype!="tmpfs"}) /
   node_filesystem_size_bytes{job="node_exporter",fstype!="tmpfs"} * 100 > 80
 
Condition: IS ABOVE 80
Evaluation: Every 1m for 5m
Annotations: Summary: "Disk usage above 80% on {{ $labels.instance }}"
```
 
### Simulate a Performance Incident
 
Install the `stress` tool to generate artificial CPU and memory load, then observe the spike in Grafana:
 
```bash
# On rhel-node
sudo dnf install stress -y
stress --cpu 2 --vm 1 --vm-bytes 512M --timeout 60s
```
 
While the stress test runs, the Grafana dashboard will show a sharp spike in CPU utilization and memory pressure on the `192.168.56.10` target. This simulates the experience of responding to a monitoring alert and correlating it to a specific workload or process.
 
---
 
## 9. Container Management with Docker
 
Docker was installed on the Ubuntu node to practice container lifecycle management, log inspection, and deployment using Docker Compose. The Prometheus and Grafana stack from the previous section is replicated here as a Compose deployment, demonstrating the difference between manual binary installation and container-based deployment.
 
### Install Docker on Ubuntu Node
 
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
 
# Add current user to docker group to avoid sudo on every command
sudo usermod -aG docker $USER
newgrp docker
```
 
### Core Docker Operations
 
```bash
# Pull and run an Nginx container
docker run -d --name webserver -p 8080:80 nginx
 
# Verify it's running
docker ps
 
# View container logs
docker logs webserver
 
# Execute a command inside a running container
docker exec -it webserver bash
 
# Inspect container metadata
docker inspect webserver
 
# Stop and remove
docker stop webserver
docker rm webserver
```
 
### Docker Compose — Monitoring Stack
 
This Compose file deploys Prometheus and Grafana as containers, replacing the manual binary installation approach from Section 8 and demonstrating a more production-realistic deployment pattern:
 
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
          - "192.168.56.10:9100"   # rhel-node (node_exporter still runs as a service)
          - "192.168.56.20:9100"   # ubuntu-node
```
 
Create `docker-compose.yml`:
 
```yaml
version: "3.8"
 
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped
 
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    restart: unless-stopped
 
volumes:
  prometheus_data:
  grafana_data:
```
 
```bash
docker compose up -d
docker compose ps
docker compose logs prometheus
docker compose logs grafana
```
 
### Troubleshooting Scenario — Container Restarting in a Loop
 
**Symptom:** `docker ps` shows a container repeatedly restarting with a short uptime.
 
**Diagnosis:**
 
```bash
# Identify the restarting container
docker ps -a
 
# View logs from the last crash cycle
docker logs --tail 50 prometheus
 
# Inspect the container's restart policy and exit code
docker inspect prometheus | grep -A5 "RestartPolicy"
docker inspect prometheus | grep "ExitCode"
```
 
**Root Cause:** A misconfigured volume mount caused Prometheus to look for its config file at the wrong path inside the container, exiting immediately with a fatal error.
 
**Resolution:**
 
```bash
# Stop and remove the broken container
docker compose down
 
# Verify the local prometheus.yml path is correct and the file exists
ls -l ./prometheus.yml
 
# Redeploy
docker compose up -d
 
# Confirm stable status
docker compose ps
docker logs prometheus --tail 20
```
 
---
 
## 10. Configuration Management with Ansible
 
Ansible was installed on the Ubuntu node as the control node, managing both lab nodes over SSH. Playbooks automate tasks that were performed manually throughout this lab — user creation, package installation, SSH hardening, and service enforcement — demonstrating the shift from manual administration to repeatable, idempotent configuration management.
 
### Install Ansible on Ubuntu Node
 
```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```
 
### Configure Inventory
 
```bash
mkdir -p ~/ansible && cd ~/ansible
vim inventory.ini
```
 
```ini
[rhel]
rhel-node ansible_host=192.168.56.10 ansible_user=alice ansible_port=2222
 
[ubuntu]
ubuntu-node ansible_host=192.168.56.20 ansible_user=alice ansible_port=2222 ansible_connection=local
 
[lab:children]
rhel
ubuntu
```
 
Test connectivity to both nodes:
 
```bash
ansible all -i inventory.ini -m ping
```
 
Both nodes should return `pong`.
 
### Playbook 1 — User and Group Management
 
Automates the user/group creation from Section 2 across both nodes simultaneously:
 
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
 
    - name: Set password aging policy (max 90 days, warn 14 days)
      command: chage -M 90 -W 14 {{ item.name }}
      loop: "{{ lab_users }}"
      changed_when: false
```
 
```bash
ansible-playbook -i inventory.ini playbooks/users.yml
```
 
### Playbook 2 — Package Installation
 
Installs a defined package list on each distribution, using `when` conditionals to handle the `dnf` vs `apt` difference:
 
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
      when: ansible_os_family == "RedHat"
 
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
      when: ansible_os_family == "Debian"
```
 
```bash
ansible-playbook -i inventory.ini playbooks/packages.yml
```
 
### Playbook 3 — SSH Hardening Enforcement
 
Deploys the hardened `sshd_config` from Section 3 to both nodes and restarts the SSH service. Uses a handler so the service only restarts if the config actually changed:
 
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
      when: ansible_os_family == "RedHat"
 
    - name: Restart SSH service (Ubuntu)
      systemd:
        name: ssh
        state: restarted
      when: ansible_os_family == "Debian"
 
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
 
    - name: Validate sshd config syntax before applying
      command: sshd -t
      changed_when: false
```
 
```bash
ansible-playbook -i inventory.ini playbooks/ssh_hardening.yml
```
 
### Playbook 4 — Service Enforcement
 
Ensures critical services are running and enabled across both nodes, reflecting how configuration management tools enforce a desired state rather than assume a prior state:
 
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
      when: ansible_os_family == "RedHat"
 
    - name: Ensure Ubuntu-specific services are running
      systemd:
        name: "{{ item }}"
        state: started
        enabled: true
      loop: "{{ ubuntu_services }}"
      when: ansible_os_family == "Debian"
```
 
```bash
ansible-playbook -i inventory.ini playbooks/services.yml
```
 
### Troubleshooting Scenario — Package Name Mismatch Between Distributions
 
**Symptom:** A playbook that installs packages fails on the RHEL node with `No package httpd-tools available` after succeeding on Ubuntu, where the same tool is named `apache2-utils`.
 
**Diagnosis:**
 
```bash
# Run with verbose output to see the exact error
ansible-playbook -i inventory.ini playbooks/packages.yml -v
 
# Confirm package names per distribution
ansible rhel -i inventory.ini -m command -a "dnf info httpd-tools"
ansible ubuntu -i inventory.ini -m command -a "apt show apache2-utils"
```
 
**Root Cause:** The playbook used a single package list for all nodes without accounting for the distribution-specific naming difference between `dnf` and `apt` package repos.
 
**Resolution:** Split the package task into distribution-specific blocks using `when: ansible_os_family`:
 
```yaml
- name: Install web utilities on RHEL
  dnf:
    name: httpd-tools
    state: present
  when: ansible_os_family == "RedHat"
 
- name: Install web utilities on Ubuntu
  apt:
    name: apache2-utils
    state: present
  when: ansible_os_family == "Debian"
```
 
---
 
## 11. Bash Automation Scripts
 
Scripts were developed on the Ubuntu node and structured to reflect enterprise operational runbook practices — including timestamped logging, configurable retention policies, and syslog integration via `logger`. The backup script pulls from both nodes over SSH.
 
### Backup Script
 
Backs up specified directories, timestamps the archive, enforces a 7-day retention policy, and pulls a remote archive from rhel-node via SSH:
 
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
 
ssh "${REMOTE_USER}@${REMOTE_HOST}" -p 2222 "mkdir -p ${REMOTE_BACKUP_DIR}"
scp -P 2222 "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BACKUP_DIR}/*.tar.gz" \
  "$BACKUP_DIR/" 2>/dev/null
logger -t "$LOG_TAG" "Remote backup pulled from ${REMOTE_HOST}"
 
find "$BACKUP_DIR" -name "*.tar.gz" -type f -mtime +$RETENTION_DAYS -exec rm {} \;
logger -t "$LOG_TAG" "Old backups older than $RETENTION_DAYS days removed"
 
echo "Backup complete: $FILENAME"
```
 
### System Health Check Script
 
Monitors CPU, memory, and disk usage across both nodes and logs alerts when thresholds are exceeded:
 
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
 
### Log Rotation Script
 
Compresses and archives logs older than 7 days, removing archives older than 30 days:
 
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
 
## 12. Troubleshooting Scenarios
 
Real failure scenarios encountered and reproduced during the lab, documenting diagnosis steps and resolutions to simulate production incident response. Scenarios marked **[RHEL]** were reproduced specifically on the RHEL node as part of RHCSA exam preparation.
 
---
 
### Scenario 1 — Service Fails to Start [RHEL]
 
**Symptom:** `sudo systemctl start myapp.service` fails with no clear error.
 
**Diagnosis:**
 
```bash
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -n 50
sudo journalctl -xe
```
 
**Root Cause:** Script path in `ExecStart` was incorrect, or the app file had a permission issue. On RHEL, SELinux context mismatches can block service execution even when file permissions appear correct.
 
**Resolution:**
 
```bash
ls -l /opt/myapp/app.py
sudo chmod 755 /opt/myapp/app.py
sudo chown bob:bob /opt/myapp/app.py
 
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
 
**Root Cause:** Syntax error in `sshd_config`, or on RHEL, SELinux blocking the new port after a port change without updating policy.
 
**Resolution:**
 
```bash
sudo vim /etc/ssh/sshd_config
sudo semanage port -a -t ssh_port_t -p tcp 2222   # RHEL only
sudo systemctl restart sshd
ssh -p 2222 alice@192.168.56.10
```
 
---
 
### Scenario 3 — Disk Space Exhaustion
 
**Symptom:** Services begin failing; logs stop writing.
 
**Diagnosis:**
 
```bash
df -h
du -sh /* 2>/dev/null | sort -rh | head -20
du -sh /var/log/* | sort -rh | head -10
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
 
### Scenario 4 — fail2ban Blocking Legitimate User
 
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
 
### Scenario 5 — SELinux Blocking Service Write [RHEL]
 
**Symptom:** A service starts successfully but cannot write to its log directory despite correct file permissions.
 
**Diagnosis:**
 
```bash
sudo ausearch -m avc -ts recent
sudo cat /var/log/audit/audit.log | grep denied
```
 
**Root Cause:** The service's target directory had the wrong SELinux file context.
 
**Resolution:**
 
```bash
ls -Z /opt/myapp/
sudo semanage fcontext -a -t var_log_t "/opt/myapp/logs(/.*)?"
sudo restorecon -Rv /opt/myapp/logs/
ls -Z /opt/myapp/logs/
```
 
---
 
### Scenario 6 — rsyslog Forwarding Silently Breaks After Firewall Reload
 
Documented in [Section 7](#7-centralized-logging-with-rsyslog).
 
---
 
### Scenario 7 — Docker Container Restarting in a Loop
 
Documented in [Section 9](#9-container-management-with-docker).
 
---
 
### Scenario 8 — Ansible Package Name Mismatch Between Distributions
 
Documented in [Section 10](#10-configuration-management-with-ansible).
 
---
 
## 13. RHCSA Exam Practice
 
This lab served as the primary preparation environment for the RHCSA exam (EX200), passed in April 2026. All tasks below were practiced on the RHEL node and reflect real exam-style objectives.
 
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
 
| Tool               | Purpose                                                             |
|--------------------|---------------------------------------------------------------------|
| journalctl         | Service and system log analysis                                     |
| systemctl          | Service management and desired state enforcement                    |
| ss                 | Active connection and socket inspection                             |
| top / htop         | Real-time process and CPU/memory monitoring                         |
| df / du            | Disk usage analysis                                                 |
| strace             | System call tracing for permission and I/O debugging                |
| firewall-cmd       | Firewall rule management (firewalld)                                |
| fail2ban-client    | Intrusion prevention status and ban management                      |
| chage              | Password aging policy enforcement                                   |
| visudo             | Safe sudoers policy editing                                         |
| ausearch           | SELinux audit log analysis                                          |
| semanage           | SELinux policy management (ports, file contexts)                    |
| restorecon         | Restore default SELinux file contexts                               |
| nmcli              | Network interface configuration (RHEL)                              |
| rsyslog            | Centralized log collection and forwarding                           |
| Prometheus         | Metrics collection and time-series storage                          |
| Node Exporter      | System-level metrics exposure for Prometheus                        |
| Grafana            | Metrics visualization, dashboards, and alerting                     |
| Docker / Compose   | Container lifecycle management and multi-service deployment         |
| Ansible            | Agentless configuration management and playbook automation          |
| stress             | Artificial CPU/memory load generation for monitoring simulation     |
| logger             | Write entries to syslog from shell scripts                          |
| /var/log analysis  | Manual log inspection across both nodes                             |
 
---
 
[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:kennymiranda000@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/kenneth-miranda-xyz)
