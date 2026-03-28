# Step 5: Wazuh Rules Reference

## Authentication & Brute-Force Rules

| Rule ID | Level | Description | Trigger |
|---------|-------|-------------|---------|
| 5760 | 5 | sshd: authentication failed | Each individual failed SSH login attempt |
| 5503 | 5 | PAM: User login failed | System-level (PAM) confirmation of login failure |
| 5551 | 10 | PAM: Multiple failed logins in a small period | Pattern detection — multiple failures from same source |
| 5720 | 10 | sshd: Multiple authentication failures | Correlation rule — sustained failed logins from same IP |
| 5763 | 10 | sshd: brute force trying to get access | High-confidence brute-force attack confirmed |

### How brute-force detection works:

```
Single failed login → Rule 5760 (Level 5)
                    → Rule 5503 (Level 5)
                         ↓ (multiple failures detected)
Pattern detected   → Rule 5551 (Level 10)
                   → Rule 5720 (Level 10)
                         ↓ (sustained attack confirmed)
Attack confirmed   → Rule 5763 (Level 10)
                         ↓ (triggers active response)
Auto-block         → Rule 651  (Level 3) — firewall-drop executed
```

## File Integrity Monitoring (FIM) Rules

| Rule ID | Level | Description | Trigger |
|---------|-------|-------------|---------|
| 550 | 7 | Integrity checksum changed | A monitored file was modified |
| 553 | 7 | File deleted | A file was removed from a monitored directory |
| 554 | 5 | File added to the system | A new file was created in a monitored directory |

### Why FIM matters:

FIM catches **post-exploitation** activity:
- Attacker drops a backdoor → Rule 554
- Attacker modifies system config → Rule 550
- Attacker deletes logs to cover tracks → Rule 553

## Active Response Rules

| Rule ID | Level | Description | Trigger |
|---------|-------|-------------|---------|
| 651 | 3 | Host Blocked by firewall-drop | Active Response executed — attacker IP blocked via iptables |

## System Health Rules

| Rule ID | Level | Description | Trigger |
|---------|-------|-------------|---------|
| 501 | 3 | New ossec agent connected | A new Wazuh agent registered with the manager |
| 502 | 3 | Ossec agent started | An agent service was started |
| 503 | 3 | Ossec agent disconnected | An agent lost connection to the manager |
| 510 | 7 | Host-based anomaly detection (rootcheck) | Rootcheck found a potential issue |
| 52002 | 3 | Apparmor DENIED | AppArmor blocked an application action |

## MITRE ATT&CK Mapping

Several rules map to MITRE ATT&CK techniques:

| Rule | MITRE Technique | Tactic |
|------|----------------|--------|
| 5760, 5763 | T1110 (Brute Force) | Credential Access |
| 5760 | T1021.004 (Remote Services: SSH) | Lateral Movement |
| 550, 554 | T1565 (Data Manipulation) | Impact |
| 651 | — | Defense / Active Response |
