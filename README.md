# 🛡️ Automated Threat Detection and Response using Open-Source SIEM

A fully functional Security Information and Event Management (SIEM) system built entirely with open-source tools. Utilizing Wazuh and TheHive, this automated SOC pipeline features real-time threat detection, Active Response (automated firewall blocking), File Integrity Monitoring (FIM), and SOAR integration for incident ticketing.

# 🛡️ Automated Threat Detection and Response using Open-Source SIEM

A fully functional **Security Information and Event Management (SIEM)** system with automated threat detection, active response, file integrity monitoring, and SOAR integration — built entirely with open-source tools.

## 📋 Project Overview

This project demonstrates an end-to-end security monitoring pipeline:

1. **Wazuh SIEM** detects brute-force attacks and file tampering in real-time
2. **Active Response** automatically blocks attacker IPs via firewall rules
3. **File Integrity Monitoring (FIM)** catches post-exploitation file modifications
4. **TheHive Integration** forwards high-severity alerts for incident case management

Built as a Capstone Project for B.Tech (Cyber Security & Digital Forensics) at VIT Bhopal University.

## 🏗️ Architecture

```
┌─────────────────┐         SSH Brute-Force        ┌─────────────────────────┐
│   Kali Linux    │ ──────────────────────────────▶ │     Ubuntu Server       │
│   (Attacker)    │         Nmap Scans              │   (Wazuh Server)        │
│   10.0.2.5      │                                 │   10.0.2.4              │
│                 │                                 │                         │
│  • Hydra        │    ◀── Firewall Block ────────  │  • Wazuh Manager        │
│  • Nmap         │         (Active Response)       │  • Wazuh Indexer        │
│  • Wazuh Agent  │                                 │  • Wazuh Dashboard      │
└─────────────────┘                                 │  • TheHive (Docker)     │
                                                    │  • Cassandra (Docker)   │
                                                    │  • Elasticsearch(Docker)│
                                                    └─────────────────────────┘
```

**Network:** VirtualBox NAT Network (`10.0.2.0/24`)

## 🔧 Tech Stack

| Component | Tool | Purpose |
|-----------|------|---------|
| SIEM Platform | Wazuh 4.7.5 | Log analysis, threat detection, alerting |
| Endpoint Agent | Wazuh Agent | Collects logs from monitored endpoints |
| Virtualization | Oracle VirtualBox | Isolated lab environment |
| Server OS | Ubuntu Server 22.04 | Hosts Wazuh server components |
| Attacker OS | Kali Linux | Attack simulation endpoint |
| Incident Response | TheHive 5.2 | Case management & alert tracking |
| Database | Cassandra 4.0 | TheHive backend storage |
| Search Engine | Elasticsearch 7.17.9 | TheHive indexing |
| Attack Tools | Hydra, Nmap | Brute-force & port scan simulation |

## 📁 Repository Structure

```
├── README.md                          # This file
├── config/
│   ├── ossec.conf                     # Wazuh Manager configuration (key sections)
│   └── custom-thehive                 # TheHive integration script
├── docs/
│   ├── 01-virtualbox-setup.md         # VirtualBox & VM setup guide
│   ├── 02-wazuh-installation.md       # Wazuh server & agent installation
│   ├── 03-thehive-setup.md            # TheHive Docker setup & integration
│   ├── 04-attack-simulation.md        # Running attacks & viewing results
│   └── 05-rules-reference.md          # Wazuh rules explained
└── screenshots/                       # Add your demo screenshots here
    └── .gitkeep
```

## 🚀 Quick Start

### Prerequisites
- **Host Machine:** Minimum 8GB RAM, 50GB free disk space
- **Software:** Oracle VirtualBox 7.x installed
- **ISOs:** Ubuntu Server 22.04, Kali Linux (latest)

### Step 1: VirtualBox Network Setup

1. Open VirtualBox → File → Tools → Network Manager
2. Go to **NAT Networks** tab → Click **Create**
3. Configure:
   - Name: `WazuhLabNet`
   - IPv4 Prefix: `10.0.2.0/24`
   - Enable DHCP: ✅

### Step 2: Create Virtual Machines

**Ubuntu Server VM (Wazuh Server):**
- RAM: 4096 MB (minimum)
- CPU: 2 cores
- Disk: 30 GB
- Network: Attach to `WazuhLabNet`

**Kali Linux VM (Attacker/Endpoint):**
- RAM: 2048 MB
- CPU: 2 cores
- Disk: 25 GB
- Network: Attach to `WazuhLabNet`

### Step 3: Install Wazuh Server (Ubuntu VM)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Download and run Wazuh installer
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a

# Save the credentials shown at the end!
# Access dashboard at https://<ubuntu-ip>
```

Verify all services:
```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

### Step 4: Install Wazuh Agent (Kali VM)

1. Open Wazuh Dashboard → Agents → Deploy New Agent
2. Select Linux DEB, enter server IP (`10.0.2.4`)
3. Copy and run the generated command on Kali:

```bash
# Example (your command will differ):
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
sudo WAZUH_MANAGER='10.0.2.4' dpkg -i wazuh-agent_4.7.5-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Verify connection:
```bash
sudo systemctl status wazuh-agent
```

### Step 5: Configure Active Response & FIM

Edit the Wazuh Manager config:
```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add these blocks (see `config/ossec.conf` for full reference):

**Active Response — Auto-block brute-force attackers:**
```xml
<active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>5720,5763</rules_id>
    <timeout>600</timeout>
</active-response>
```

**Active Response — Auto-block port scanners:**
```xml
<active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>581</rules_id>
    <timeout>600</timeout>
</active-response>
```

**File Integrity Monitoring — Real-time detection:**
```xml
<directories realtime="yes">/etc,/usr/bin,/usr/sbin</directories>
```

Restart after changes:
```bash
sudo systemctl restart wazuh-manager
```

### Step 6: Setup TheHive (Docker)

```bash
# Install Docker
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker

# Start Cassandra
sudo docker run -d --name cassandra \
  -p 9042:9042 \
  -e CASSANDRA_CLUSTER_NAME=TheHive \
  cassandra:4.0

# Wait 60 seconds for Cassandra to initialize
sleep 60

# Start Elasticsearch (use port 9201 to avoid conflict with Wazuh Indexer on 9200)
sudo docker run -d --name elasticsearch \
  -p 9201:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:7.17.9

# Wait 30 seconds
sleep 30

# Start TheHive
sudo docker run -d --name thehive \
  -p 9000:9000 \
  strangebee/thehive:5.2

# Access TheHive at http://<ubuntu-ip>:9000
# Default login: admin@thehive.local / secret
```

### Step 7: Integrate Wazuh → TheHive

**Create the integration script:**
```bash
sudo nano /var/ossec/integrations/custom-thehive
```

Paste the contents from `config/custom-thehive`, then:
```bash
sudo chmod 750 /var/ossec/integrations/custom-thehive
sudo chown root:wazuh /var/ossec/integrations/custom-thehive
```

**Create a TheHive org and user with alert permissions:**
```bash
sudo /var/ossec/framework/python/bin/python3 -c "
import requests
headers = {
    'Authorization': 'Bearer <ADMIN_API_KEY>',
    'Content-Type': 'application/json'
}

# Create SOC organisation
r = requests.post('http://127.0.0.1:9000/api/v1/organisation',
    headers=headers,
    json={'name': 'SOC', 'description': 'Security Operations'},
    verify=False)
print('Create org:', r.status_code)

# Create analyst user in SOC org
r2 = requests.post('http://127.0.0.1:9000/api/v1/user',
    headers=headers,
    json={'login': 'soc-wazuh@thehive.local', 'name': 'Wazuh SIEM',
          'profile': 'analyst', 'organisation': 'SOC'},
    verify=False)
print('Create user:', r2.status_code)

# Generate API key
r3 = requests.post('http://127.0.0.1:9000/api/v1/user/soc-wazuh@thehive.local/key/renew',
    headers=headers, verify=False)
print('API key:', r3.text)
"
```

**Add integration to ossec.conf:**
```xml
<integration>
    <name>custom-thehive</name>
    <hook_url>http://127.0.0.1:9000/api/alert</hook_url>
    <api_key>YOUR_SOC_USER_API_KEY</api_key>
    <alert_format>json</alert_format>
    <rule_id>5720,5763</rule_id>
</integration>
```

Restart Wazuh:
```bash
sudo systemctl restart wazuh-manager
```

## ⚔️ Attack Simulations

### Attack 1: SSH Brute-Force (from Kali)

```bash
# Single password attempt (loop)
while true; do hydra -l maddy -p "wrongpassword123" ssh://10.0.2.4; sleep 1; done

# Or with wordlist
hydra -l maddy -P /usr/share/wordlists/rockyou.txt ssh://10.0.2.4 -t 4
```

**Expected Wazuh Alerts:**
| Rule ID | Level | Description |
|---------|-------|-------------|
| 5760 | 5 | sshd: authentication failed |
| 5503 | 5 | PAM: User login failed |
| 5551 | 10 | PAM: Multiple failed logins in a small period of time |
| 5763 | 10 | sshd: brute force trying to get access to the system |
| 651 | 3 | Host Blocked by firewall-drop Active Response |

**Expected TheHive Alert:** Wazuh Alert - Rule 5763 (auto-created)

### Attack 2: File Integrity Tampering (on Ubuntu)

```bash
# Simulate attacker dropping a backdoor
sudo touch /etc/malicious_backdoor

# Simulate attacker modifying system files
sudo bash -c 'echo "compromised" >> /etc/hosts'
```

**Expected Wazuh Alerts:**
| Rule ID | Level | Description |
|---------|-------|-------------|
| 554 | 5 | File added to the system |
| 550 | 7 | Integrity checksum changed |
| 553 | 7 | File deleted (if you delete the test file) |

## 🔑 Wazuh Rules Reference

| Rule ID | Category | Description |
|---------|----------|-------------|
| 5760 | Authentication | Individual SSH login failure |
| 5503 | Authentication | PAM-level login failure confirmation |
| 5551 | Brute-Force | Multiple failed logins detected (pattern) |
| 5720 | Brute-Force | Multiple authentication failures from same IP |
| 5763 | Brute-Force | Brute-force attack confirmed |
| 581 | Port Scan | Host-based anomaly detection — connection attempt (port scan) |
| 651 | Active Response | Attacker IP blocked by firewall-drop |
| 550 | File Integrity | File checksum changed (modification) |
| 554 | File Integrity | New file added to monitored directory |
| 553 | File Integrity | File deleted from monitored directory |

## 🧹 Cleanup After Demo

```bash
# Remove test files
sudo rm /etc/malicious_backdoor /etc/hacker_was_here

# Remove added lines from /etc/hosts
sudo nano /etc/hosts  # Delete lines containing "hacked", "compromised", etc.

# Unblock attacker IP (or wait 600 seconds for auto-unblock)
sudo /var/ossec/active-response/bin/firewall-drop delete - - 10.0.2.5
```

## 🔮 Future Enhancements

- **Cortex Integration:** Automated threat intelligence enrichment (IP reputation, MISP lookups)
- **MITRE ATT&CK Mapping:** Full tactic/technique mapping for all detected threats
- **Email/Slack Notifications:** Real-time alert notifications
- **Custom Dashboards:** Grafana integration for security metrics visualization
- **Network IDS:** Suricata integration for network-level threat detection

## 👥 Team

| Name | Role |
|------|------|
| Madhan Mohan Naidu .M (22BCY10182) | Full system development — Wazuh server/agent setup, Active Response configuration, File Integrity Monitoring, TheHive Docker deployment, Wazuh-TheHive integration script |
| Rehan Rajesh (22BCY10196) | attack simulation & validation , presentation preparation, and documentation |
| V.A.Aaswin (22BCY10257) |Project report writing, Research and data gathering for report  |
| Insaf Sadik A S (22BCY10282) | Research and data gathering for report |

**Supervisor:** Dr. Hariharasitaraman S  
**Institution:** VIT Bhopal University

## 📄 License

This project is for educational purposes as part of the B.Tech Capstone Project (DSN4091).
