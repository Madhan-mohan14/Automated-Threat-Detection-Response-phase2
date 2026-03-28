# Step 4: Attack Simulations

## 4.1 Attack 1: SSH Brute-Force

### From Kali Linux terminal:

**Single password loop (good for demos):**
```bash
while true; do hydra -l maddy -p "wrongpassword123" ssh://10.0.2.4; sleep 1; done
```
Run for ~30 seconds, then `Ctrl+C` to stop.

**Wordlist attack (more realistic):**
```bash
hydra -l maddy -P /usr/share/wordlists/rockyou.txt ssh://10.0.2.4 -t 4
```

### What to observe:

**In Kali terminal:**
- Multiple "0 valid password found" messages
- Eventually: `Timeout connecting to 10.0.2.4` — this means the Active Response blocked the attacker IP

**In Wazuh Dashboard → Security Events:**
- Rule 5760 (Level 5): `sshd: authentication failed` — each failed attempt
- Rule 5503 (Level 5): `PAM: User login failed` — system-level confirmation
- Rule 5551 (Level 10): `PAM: Multiple failed logins in a small period of time`
- Rule 5763 (Level 10): `sshd: brute force trying to get access to the system`
- Rule 651 (Level 3): `Host Blocked by firewall-drop Active Response`

**In TheHive → Alerts:**
- "Wazuh Alert - Rule 5763" automatically created

### Unblocking the attacker IP (for re-testing):
The block auto-expires after 600 seconds (10 minutes). To unblock immediately:
```bash
# On Ubuntu server
sudo /var/ossec/active-response/bin/firewall-drop delete - - 10.0.2.5
```

## 4.2 Attack 2: File Integrity Tampering

### On Ubuntu server terminal:

**Create a suspicious file:**
```bash
sudo touch /etc/malicious_backdoor
```

**Modify a critical system file:**
```bash
sudo bash -c 'echo "compromised" >> /etc/hosts'
```

### What to observe:

**In Wazuh Dashboard → Integrity Monitoring → Events:**
- Rule 554 (Level 5): `File added to the system` — for `/etc/malicious_backdoor`
- Rule 550 (Level 7): `Integrity checksum changed` — for `/etc/hosts`

The alerts show:
- Exact file path that was changed
- Type of event (added/modified/deleted)
- Timestamp of the change
- Agent that detected it

### Cleanup after testing:
```bash
sudo rm /etc/malicious_backdoor
sudo nano /etc/hosts  # Remove the "compromised" line
```

## 4.3 Demo Flow for Video Recording

### Before recording:
```bash
# Verify Wazuh is running
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard

# Start TheHive stack
sudo docker start cassandra && sleep 30
sudo docker start elasticsearch && sleep 20
sudo docker start thehive
# Wait 2 minutes for TheHive to fully start
```

### Recording sequence:

1. **Show clean state**
   - Wazuh Dashboard: Security Events (empty/minimal)
   - TheHive: Alerts page (empty)
   - Both VMs running

2. **Brute-force attack** (from Kali)
   ```bash
   while true; do hydra -l maddy -p "wrongpassword123" ssh://10.0.2.4; sleep 1; done
   ```
   - Show Wazuh alerts appearing in real-time
   - Show Rule 651 (auto-block)
   - Show TheHive alert created automatically
   - Click alert → Create Case

3. **File tampering** (on Ubuntu)
   ```bash
   sudo touch /etc/malicious_backdoor
   sudo bash -c 'echo "compromised" >> /etc/hosts'
   ```
   - Show Integrity Monitoring → Events
   - Show Rule 550 and 554

4. **Conclusion**
   - Recap: Two attack types detected
   - Automated blocking demonstrated
   - SOAR integration with TheHive working
   - Mention future work: Cortex, MITRE mapping
