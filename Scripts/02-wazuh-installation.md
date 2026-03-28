# Step 2: Wazuh Installation

## 2.1 Install Wazuh Server (Ubuntu VM)

### Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### Run the Wazuh all-in-one installer
This installs Manager, Indexer, and Dashboard in one command:
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

**Important:** Save the admin credentials shown at the end of installation. You'll need them to log into the dashboard.

### Verify all services are running
```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

All three should show `active (running)`.

### Access the Dashboard
Open a browser on the Ubuntu VM and go to:
```
https://127.0.0.1
```

Login with the credentials from the installation output.

## 2.2 Install Wazuh Agent (Kali VM)

### From the Wazuh Dashboard
1. Go to **Agents** → **Deploy new agent**
2. Select:
   - Operating system: **DEB amd64**
   - Server address: `10.0.2.4`
   - Agent name: `kali-linux`
3. Copy the generated command

### Run on Kali Linux
Paste and run the generated command. It will look similar to:
```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
sudo WAZUH_MANAGER='10.0.2.4' dpkg -i wazuh-agent_4.7.5-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Verify agent connection
```bash
sudo systemctl status wazuh-agent
```

Check the Wazuh Dashboard → **Agents** section. The Kali agent should appear with **Active** status.

## 2.3 Configure Active Response

Edit the Wazuh Manager configuration:
```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the active response block before `</ossec_config>`:
```xml
<active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>5720,5763</rules_id>
    <timeout>600</timeout>
</active-response>
```

This tells Wazuh to automatically run `firewall-drop` (iptables block) for 600 seconds whenever rules 5720 or 5763 are triggered.

## 2.4 Configure File Integrity Monitoring

In the same `ossec.conf`, find the `<syscheck>` section and update the directories line:

**Change:**
```xml
<directories>/etc,/usr/bin,/usr/sbin</directories>
```

**To:**
```xml
<directories realtime="yes">/etc,/usr/bin,/usr/sbin</directories>
```

The `realtime="yes"` attribute enables instant detection instead of waiting for periodic scans.

### Restart Wazuh Manager
```bash
sudo systemctl restart wazuh-manager
```
