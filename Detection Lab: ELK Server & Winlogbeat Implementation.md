# Detection Lab: ELK Server & Winlogbeat Implementation

This document details the architecture, installation, and configuration of a self-managed Elastic Stack (ELK) server on Ubuntu ARM64, paired with Winlogbeat on a Windows 11 ARM endpoint. It also explains the persistence mechanisms that guarantee continuous log shipping.

## Table of Contents
- [1. Architecture Overview](#1-architecture-overview)
- [2. ELK Server Implementation (Ubuntu ARM64)](#2-elk-server-implementation-ubuntu-arm64)
- [3. Windows 11 ARM Endpoint Configuration](#3-windows-11-arm-endpoint-configuration)
- [4. Winlogbeat Installation & Configuration](#4-winlogbeat-installation--configuration)
- [5. Persistence Mechanisms](#5-persistence-mechanisms)
- [6. Verification & ES|QL Queries](#6-verification--esql-queries)



## 1. Architecture Overview

| Component | OS / Platform | Role |
| :--- | :--- | :--- |
| **ELK Server** | Ubuntu Server (ARM64) | Hosts Elasticsearch, Kibana, and the Security App. |
| **Endpoint** | Windows 11 (ARM64) | Hosts Winlogbeat, Atomic Red Team, and generates telemetry. |
| **Network** | VMware Fusion NAT / Host-Only | Isolated lab network (e.g., `192.168.212.x`). |

---

## 2. ELK Server Implementation (Ubuntu ARM64)

### 2.1. Elasticsearch Installation & Security
Elasticsearch 8.x enables security by default. To run it as a self-managed SIEM backend, Transport SSL must be configured.

**1. Install Elasticsearch:**
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
sudo apt-get install apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install elasticsearch
```

**2. Generate Transport Certificates:**
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca --out /etc/elasticsearch/elastic-stack-ca.p12 --pass ""
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca /etc/elasticsearch/elastic-stack-ca.p12 --ca-pass "" --out /etc/elasticsearch/elastic-certificates.p12 --pass ""

sudo chown elasticsearch:elasticsearch /etc/elasticsearch/*.p12
sudo chmod 600 /etc/elasticsearch/*.p12
```

**3. Configure `elasticsearch.yml`:**
```yaml
cluster.name: elk-arm-cluster
node.name: node-1
network.host: 0.0.0.0
discovery.type: single-node

# Security Settings
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/elastic-certificates.p12
```

**4. Start and Set Passwords:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch

# Reset the 'elastic' superuser password
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -i

# Reset the 'kibana_system' password (used by Kibana backend)
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -i
```

### 2.2. Kibana Installation & Encryption Keys
To use the Elastic Security app and prebuilt detection rules, Kibana requires an encrypted saved objects key.

**1. Install Kibana:**
```bash
sudo apt-get install kibana
```

**2. Generate Encryption Key:**
```bash
openssl rand -hex 32
# Output example: 871ee80894b883ca4f69c515ed8d37a96a70c38820a17f1b32cef98d43f57826
```

**3. Configure `kibana.yml`:**
```yaml
server.host: "0.0.0.0"
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "YOUR_KIBANA_SYSTEM_PASSWORD"
elasticsearch.ssl.certificateAuthorities: ["/etc/elasticsearch/elastic-stack-ca.p12"]

# REQUIRED for Elastic Security Detections
xpack.encryptedSavedObjects.encryptionKey: "YOUR_GENERATED_32_CHAR_HEX_KEY"
```

**4. Start Kibana:**
```bash
sudo systemctl enable --now kibana
```

---

## 3. Windows 11 ARM Endpoint Configuration

Before installing Winlogbeat, Windows must be configured to log process command lines. By default, Windows Event ID 4688 does not include the arguments passed to executables.

**1. Enable Command Line Auditing via Registry:**
Run this in an **Administrator PowerShell** window:
```powershell
New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Name "ProcessCreationIncludeCmdLine_Enabled" -Value 1 -Type DWord -Force
```

**2. Reboot the VM:**
Windows 11 requires a reboot for this kernel-level audit policy to take effect.
```powershell
Restart-Computer -Force
```

---

## 4. Winlogbeat Installation & Configuration

Because Elastic does not publish a native Windows ARM64 build of Winlogbeat, we use the **x86_64 build**, which runs flawlessly under Windows 11 ARM's built-in x64 emulation layer.

### 4.1. Installation
```powershell
# Download Winlogbeat 8.x (Match your ELK server version)
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-8.15.2-windows-x86_64.zip" -OutFile "$env:TEMP\winlogbeat.zip"

# Extract and Install
Expand-Archive "$env:TEMP\winlogbeat.zip" -DestinationPath "C:\Program Files"
Rename-Item "C:\Program Files\winlogbeat-8.15.2-windows-x86_64" "C:\Program Files\Winlogbeat"

cd 'C:\Program Files\Winlogbeat'
.\install-service-winlogbeat.ps1
```

### 4.2. Configuration (`winlogbeat.yml`)
Replace the contents of `C:\Program Files\Winlogbeat\winlogbeat.yml` with the following:

```yaml
winlogbeat.event_logs:
  - name: Security
    # Captures Process Creation (4688), Logons, Log Clearing, Scheduled Tasks
  - name: System
    # Captures Service Creation (7045)
  - name: Microsoft-Windows-PowerShell/Operational
    event_id: 4103, 4104, 4105, 4106
    # Captures Script Block Logging (4104)

output.elasticsearch:
  hosts: ["YOUR_ELASTIC_IP:9200"]
  username: "elastic"
  password: "YOUR_ELASTIC_SUPERUSER_PASSWORD"
  ssl.verification_mode: "none" # Acceptable for isolated lab environments

setup.kibana:
  host: "YOUR_ELASTIC_IP:5601"
  username: "elastic"
  password: "YOUR_ELASTIC_SUPERUSER_PASSWORD"
```

### 4.3. Load Ingest Pipelines and Dashboards
Run this command **once** in an Administrator PowerShell to push the official Elastic parsing pipelines and Kibana dashboards to your server:

```powershell
.\winlogbeat.exe setup -e
```

### 4.4. Start the Service
```powershell
Start-Service winlogbeat
```

---

## 5. Persistence Mechanisms

Winlogbeat guarantees continuous, reliable log shipping through two distinct persistence mechanisms:

### 5.1. State Persistence (The Registry File)
Winlogbeat maintains a local SQLite-style registry file located at `C:\Program Files\Winlogbeat\data\registry\filebeat\`. 
* **Mechanism:** Every time Winlogbeat reads an event from the Windows Event Log API, it records the `record_number` (or bookmark) of that event into the registry file.
* **Fault Tolerance:** If the Windows VM reboots, or if the Winlogbeat process crashes, the service will restart, read the registry file, and resume reading the Windows Event Logs **exactly where it left off**. No logs are lost or duplicated.

### 5.2. Service Persistence (Windows Service Manager)
The `install-service-winlogbeat.ps1` script registers Winlogbeat as a native Windows Service.
* **Mechanism:** It is configured with an `Automatic` startup type.
* **Survival:** It runs under the `NT AUTHORITY\SYSTEM` account, meaning it starts before any user logs in, survives user logoffs, and has the necessary privileges to read restricted Security event logs.

---

## 6. Verification & ES|QL Queries

Once Winlogbeat is running, verify telemetry is flowing into Kibana Discover. Use **ES|QL** (Elasticsearch Query Language) for precise, piped filtering.

### Detection 1: Suspicious Encoded PowerShell (T1059.001)
```esql
FROM logs-*
| WHERE process.name == "powershell.exe"
| WHERE process.command_line LIKE "*-enc*" OR process.command_line LIKE "*-EncodedCommand*"
| SORT @timestamp DESC
| KEEP @timestamp, host.name, user.name, process.parent.name, process.command_line
| LIMIT 100
```

### Detection 2: Scheduled Task Creation (T1053.005)
```esql
FROM logs-*
| WHERE event.code == "4698"
| WHERE winlog.event_data.TaskName LIKE "*Atomic*" OR winlog.event_data.TaskName LIKE "*Update*"
| SORT @timestamp DESC
| KEEP @timestamp, host.name, winlog.event_data.TaskName, winlog.event_data.TaskContent
```

### Detection 3: Windows Event Log Cleared (T1070.001)
```esql
FROM logs-*
| WHERE event.code == "1102"
| SORT @timestamp DESC
| KEEP @timestamp, host.name, user.name, message
```
```
