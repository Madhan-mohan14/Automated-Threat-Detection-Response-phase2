# Step 3: TheHive Setup & Wazuh Integration

## 3.1 Install Docker

```bash
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
```

## 3.2 Start TheHive Stack

TheHive requires Cassandra (database) and Elasticsearch (search).

**Important:** Wazuh Indexer already uses port 9200, so we use 9201 for TheHive's Elasticsearch.

### Start Cassandra
```bash
sudo docker run -d --name cassandra \
  -p 9042:9042 \
  -e CASSANDRA_CLUSTER_NAME=TheHive \
  cassandra:4.0
```
Wait 60 seconds for Cassandra to fully initialize.

### Start Elasticsearch
```bash
sudo docker run -d --name elasticsearch \
  -p 9201:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:7.17.9
```
Wait 30 seconds.

### Start TheHive
```bash
sudo docker run -d --name thehive \
  -p 9000:9000 \
  strangebee/thehive:5.2
```

### Verify
```bash
sudo docker ps
# Should show cassandra, elasticsearch, and thehive running
```

Access TheHive at `http://localhost:9000`

Default login: `admin@thehive.local` / `secret`

## 3.3 Starting TheHive After Reboot

The containers don't auto-start. Run these in order:
```bash
sudo docker start cassandra
sleep 30
sudo docker start elasticsearch
sleep 20
sudo docker start thehive
```
Wait 1-2 minutes before accessing `http://localhost:9000`.

## 3.4 Create Integration User

TheHive 5.x has a key concept: the default "admin" is a **platform admin** with limited org-level permissions. You need to create a user in a proper organization with `analyst` profile to have `manageAlert/create` permission.

### Run the setup script
```bash
sudo /var/ossec/framework/python/bin/python3 -c "
import requests

ADMIN_KEY = 'YOUR_ADMIN_API_KEY'  # Get from TheHive admin user settings
headers = {
    'Authorization': f'Bearer {ADMIN_KEY}',
    'Content-Type': 'application/json'
}

# Step 1: Create a SOC organisation
r = requests.post('http://127.0.0.1:9000/api/v1/organisation',
    headers=headers,
    json={'name': 'SOC', 'description': 'Security Operations Center'},
    verify=False)
print('Create org:', r.status_code)

# Step 2: Create integration user with analyst profile
r2 = requests.post('http://127.0.0.1:9000/api/v1/user',
    headers=headers,
    json={
        'login': 'soc-wazuh@thehive.local',
        'name': 'Wazuh SIEM',
        'profile': 'analyst',
        'organisation': 'SOC'
    },
    verify=False)
print('Create user:', r2.status_code)

# Step 3: Generate API key for integration
r3 = requests.post(
    'http://127.0.0.1:9000/api/v1/user/soc-wazuh@thehive.local/key/renew',
    headers=headers, verify=False)
print('API key:', r3.text)

# Step 4: Set password (to login via browser)
r4 = requests.post(
    'http://127.0.0.1:9000/api/v1/user/soc-wazuh@thehive.local/password/set',
    headers=headers,
    json={'password': 'wazuh1234'},
    verify=False)
print('Password set:', r4.status_code)
"
```

**Save the API key printed in Step 3** — you need it for the Wazuh integration config.

### Why this matters
- The `admin` profile has permissions like `manageOrganisation`, `managePlatform` — but NOT `manageAlert/create`
- The `analyst` profile has `manageAlert/create`, `manageCase/create`, etc.
- Creating alerts via API requires `manageAlert/create` permission
- This is the #1 gotcha when integrating Wazuh with TheHive 5.x

## 3.5 Create the Integration Script

```bash
sudo nano /var/ossec/integrations/custom-thehive
```

Paste the script from `config/custom-thehive` in this repository.

Set permissions:
```bash
sudo chmod 750 /var/ossec/integrations/custom-thehive
sudo chown root:wazuh /var/ossec/integrations/custom-thehive
```

## 3.6 Configure Wazuh Integration

Edit ossec.conf:
```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add before `</ossec_config>`:
```xml
<integration>
    <name>custom-thehive</name>
    <hook_url>http://127.0.0.1:9000/api/alert</hook_url>
    <api_key>YOUR_SOC_USER_API_KEY</api_key>
    <alert_format>json</alert_format>
    <rule_id>5720,5763</rule_id>
</integration>
```

Replace `YOUR_SOC_USER_API_KEY` with the key from Step 3.4.

Restart Wazuh:
```bash
sudo systemctl restart wazuh-manager
```

## 3.7 Verify Integration

Check logs:
```bash
sudo grep -i "thehive\|integration" /var/ossec/logs/ossec.log | tail -10
```

You should see:
```
wazuh-integratord: INFO: Enabling integration for: 'custom-thehive'.
```

If you see `File not found inside 'integrations'`, the script wasn't created properly. Recheck step 3.5.

## 3.8 Test the Integration

```bash
sudo /var/ossec/framework/python/bin/python3 -c "
import requests
headers = {
    'Authorization': 'Bearer YOUR_SOC_USER_API_KEY',
    'Content-Type': 'application/json'
}
data = {
    'title': 'Test Alert from Wazuh',
    'description': 'Integration test',
    'type': 'external',
    'source': 'Wazuh',
    'sourceRef': 'test-001',
    'severity': 2
}
r = requests.post('http://127.0.0.1:9000/api/alert',
    headers=headers, json=data, verify=False)
print(r.status_code)  # Should be 201
print(r.text)
"
```

If you get `201`, the integration is working. Login to TheHive as `soc-wazuh@thehive.local` / `wazuh1234` to see the alert.

## 3.9 Viewing Alerts in TheHive

1. Open `http://localhost:9000`
2. Login as `soc-wazuh@thehive.local` / `wazuh1234`
3. Click the **Alerts** icon in the left sidebar (bell icon)
4. Alerts from Wazuh will appear with titles like "Wazuh Alert - Rule 5763"
5. Click an alert → **Create Case** to convert it into an investigation case

## 3.10 Troubleshooting

| Problem | Solution |
|---------|----------|
| `File not found inside 'integrations'` | Script not created at `/var/ossec/integrations/custom-thehive` |
| `403 AuthorizationError manageAlert/create` | API key belongs to admin profile user, not analyst profile |
| `Connection refused localhost:9000` | TheHive container not running — `sudo docker start thehive` |
| `Port 9200 already in use` | Wazuh Indexer using 9200 — use 9201 for TheHive's Elasticsearch |
| Alerts not showing in TheHive | Login as org-level user (analyst profile), not platform admin |
