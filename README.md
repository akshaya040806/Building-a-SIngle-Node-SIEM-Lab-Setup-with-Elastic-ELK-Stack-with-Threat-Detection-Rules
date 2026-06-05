# SIEM Live Case Lab — Elastic SIEM with Insider Threat Detection
### Build. Simulate. Detect.

---

## Project Overview

This project builds a fully functional Security Information and Event Management (SIEM) lab on a single Kali Linux VM. It simulates a corporate environment with 7 employees across 4 departments, generates realistic enterprise telemetry, ships logs via Filebeat into Elasticsearch, and detects attack patterns using Elastic SIEM detection rules in Kibana.

**Stack:**
```
log_gen.py → events_YYYY-MM-DD.jsonl → Filebeat → Elasticsearch → Kibana (Elastic SIEM)
```

**Attack types simulated:**
- Brute Force Login (external attacker, random public IP)
- Insider Threat (legitimate employee exfiltrating Finance data)
- Privilege Escalation (sudo abuse + rogue user creation)

**MITRE ATT&CK mapping:**
| Attack | Technique |
|--------|-----------|
| Brute Force | T1110 |
| Insider Threat | T1078 |
| Data Exfiltration | T1041 |
| Privilege Escalation | T1548 |

---

## System Requirements

| Component | Minimum |
|-----------|---------|
| RAM | 6GB (assigned to VM) |
| Disk | 10GB free |
| OS | Kali Linux (rolling) |
| Java | OpenJDK 17+ |
| Python | 3.10+ |

---

## Project Structure

```
~/siem/
├── log_gen.py          # Log generator — simulates corporate activity + attacks
├── log_collector.py    # Optional SQLite collector (for local analysis)
├── logs/
│   └── events_YYYY-MM-DD.jsonl   # Daily rotating log files
└── README.md
```

---

## Phase 1 — Install Elasticsearch

### Step 1 — Add Elastic GPG key
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

### Step 2 — Add Elastic repository
```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### Step 3 — Update apt and install
```bash
sudo apt update
sudo apt install elasticsearch -y
```

> **IMPORTANT:** When installation finishes, scroll up and copy the auto-generated password for the `elastic` user. It looks like:
> `The generated password for the elastic built-in superuser is: XXXXXXXXXXXXXXXXXX`
> You cannot recover this without resetting. Save it immediately.

### Step 4 — Configure Elasticsearch
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Add at the bottom:
```yaml
cluster.name: siem-lab
node.name: kali-node-1
discovery.type: single-node
xpack.ml.enabled: false
```

Also find and comment out this line:
```yaml
# cluster.initial_master_nodes: ["kali"]
```

### Step 5 — Reduce heap size (important for 6GB VM)
```bash
sudo nano /etc/elasticsearch/jvm.options.d/heap.options
```

Add:
```
-Xms512m
-Xmx512m
```

### Step 6 — Start Elasticsearch
```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

### Step 7 — Verify
```bash
curl -k -u elastic:YOUR_PASSWORD https://localhost:9200
```

Expected response: JSON with `"cluster_name": "siem-lab"`

---

## Phase 2 — Install Kibana

### Step 8 — Install
```bash
sudo apt install kibana -y
```

### Step 9 — Configure
```bash
sudo nano /etc/kibana/kibana.yml
```

Add at the bottom:
```yaml
server.port: 5601
server.host: "localhost"
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.ssl.verificationMode: "none"
```

### Step 10 — Generate encryption keys
```bash
sudo /usr/share/kibana/bin/kibana-encryption-keys generate
```

Copy the three lines printed and paste them into `/etc/kibana/kibana.yml`:
```yaml
xpack.encryptedSavedObjects.encryptionKey: <generated>
xpack.reporting.encryptionKey: <generated>
xpack.security.encryptionKey: <generated>
```

### Step 11 — Generate enrollment token
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

Copy the token. **Tokens expire in 30 minutes** — if it expires, generate a new one.

### Step 12 — Configure Kibana with token
```bash
sudo systemctl stop kibana
sudo /usr/share/kibana/bin/kibana-setup --enrollment-token PASTE_TOKEN_HERE
```

Expected output: `✔ Kibana configured successfully.`

### Step 13 — Start Kibana
```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

Wait 3-5 minutes on first start (Kibana sets up 172 plugins and internal indexes). Check when ready:
```bash
curl -s http://localhost:5601/api/status | grep -o '"level":"[^"]*"'
```

When it shows `"level":"available"` — open Firefox at `http://localhost:5601`

Login: `elastic` / `YOUR_PASSWORD`

---

## Phase 3 — Install and Configure Filebeat

### Step 14 — Install
```bash
sudo apt install filebeat -y
```

### Step 15 — Configure
```bash
sudo nano /etc/filebeat/filebeat.yml
```

Replace entire contents with:
```yaml
filebeat.inputs:
  - type: filestream
    id: siem-lab
    enabled: true
    paths:
      - /home/kali/siem/logs/events_*.jsonl
    parsers:
      - ndjson:
          target: ""
          overwrite_keys: true

processors:
  - timestamp:
      field: timestamp
      layouts:
        - '2006-01-02T15:04:05'
      target_field: '@timestamp'

output.elasticsearch:
  hosts: ["https://localhost:9200"]
  username: "elastic"
  password: "YOUR_PASSWORD"
  ssl.verification_mode: none
  index: "siem-events"
  allow_older_versions: true

setup.ilm.enabled: false
setup.template.enabled: false
```

### Step 16 — Test config
```bash
sudo filebeat test config
sudo filebeat test output
```

Both must return `OK`.

### Step 17 — Start Filebeat
```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

---

## Phase 4 — Run the Log Generator

### Step 18 — Start generator
```bash
cd /home/kali/siem
python3 log_gen.py
```

The generator simulates a full corporate workday (08:00–18:30). It produces:
- Parallel user sessions with realistic Markov-chain behaviour
- Idle gaps for meetings and lunch breaks
- Attack chains every 3 sessions in round-robin rotation

Attack rotation order:
```
Sessions 1-3  → brute_force
Sessions 4-6  → insider
Sessions 7-9  → priv_esc
Sessions 10-12 → brute_force (repeats)
```

### Step 19 — Verify events in Elasticsearch
```bash
curl -k -u elastic:YOUR_PASSWORD "https://localhost:9200/siem-events/_count"
```

Should return `"count": 1000+` after a few minutes.

---

## Phase 5 — Kibana Setup

### Step 20 — Create Data View
1. Kibana → ☰ Menu → Stack Management → Data Views
2. Create data view
3. Name: `siem-events`
4. Index pattern: `siem-events`
5. Timestamp field: `@timestamp`
6. Save

### Step 21 — Verify in Discover
1. Kibana → ☰ Menu → Discover
2. Select `siem-events` data view
3. Set time range to Last 1 year
4. You should see all events

---

## Phase 6 — Detection Rules (Elastic SIEM)

Go to: Kibana → ☰ Menu → Security → Rules → Detection Rules (SIEM)

---

### Rule 1 — Brute Force Login Detection

- Type: **Threshold**
- Index patterns: `siem-events`
- Custom query: `event_type: "login_failure"`
- Threshold field: `source_ip.keyword`
- Threshold count: `5`
- Name: `Brute Force Login Detection`
- Severity: `High`
- Risk score: `73`
- Schedule: every `5 minutes`

---

### Rule 2 — Insider Threat: Cross-Department File Access

- Type: **Query**
- Index patterns: `siem-events`
- Custom query:
```
event_type: "file_access" AND details.file: ("salary.xlsx" OR "budget.xlsx" OR "accounts.csv") AND NOT department: "Finance"
```
- Name: `Insider Threat - Unauthorised Finance File Access`
- Severity: `High`
- Risk score: `75`
- Schedule: every `5 minutes`

---

### Rule 3 — Data Exfiltration

- Type: **Query**
- Index patterns: `siem-events`
- Custom query:
```
event_type: "file_transfer" AND details.destination: "external_storage"
```
- Name: `Data Exfiltration - External File Transfer`
- Severity: `Critical`
- Risk score: `90`
- Schedule: every `5 minutes`

---

### Rule 4 — Privilege Escalation Sequence

- Type: **EQL**
- Index patterns: `siem-events`
- EQL query:
```eql
sequence by session_id [
  any where event_type == "sudo_execution"
] [
  any where event_type == "user_created"
]
```
- Name: `Privilege Escalation - Sudo to User Creation`
- Severity: `Critical`
- Risk score: `90`
- Schedule: every `5 minutes`

---

### Rule 5 — Brute Force Success (Account Compromised)

- Type: **EQL**
- Index patterns: `siem-events`
- EQL query:
```eql
sequence by session_id with maxspan=30m [
  any where event_type == "login_failure"
] [
  any where event_type == "login_success" and attack_type == "brute_force"
]
```
- Name: `Brute Force Success - Account Compromised`
- Severity: `Critical`
- Risk score: `95`
- Schedule: every `5 minutes`

---

## Troubleshooting

---

### Elasticsearch won't start — exit code 1

**Symptom:**
```
Job for elasticsearch.service failed because the control process exited with error code.
```

**Fix 1 — cluster.initial_master_nodes conflict:**
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```
Comment out:
```yaml
# cluster.initial_master_nodes: ["kali"]
```

**Fix 2 — Heap too large:**
```bash
sudo nano /etc/elasticsearch/jvm.options.d/heap.options
```
Set:
```
-Xms512m
-Xmx512m
```

**Fix 3 — Unknown setting error:**
Check the exact error:
```bash
sudo -u elasticsearch /usr/share/elasticsearch/bin/elasticsearch 2>&1 | tail -20
```
Remove any invalid settings from `elasticsearch.yml`.

---

### Kibana takes too long to start

This is normal on first start only. Kibana creates internal Elasticsearch indexes for 172 plugins. On a 6GB VM this takes 3-5 minutes.

Check progress:
```bash
curl -s http://localhost:5601/api/status | grep -o '"level":"[^"]*"'
```

Only proceed when it shows `"level":"available"`.

Do NOT restart Kibana while it is loading — this resets the process.

---

### Kibana enrollment token expired

**Symptom:**
```
API key is expired
```

**Fix:** Generate a new token (they expire in 30 minutes):
```bash
sudo systemctl stop kibana
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
sudo /usr/share/kibana/bin/kibana-setup --enrollment-token NEW_TOKEN
sudo systemctl start kibana
```

---

### Kibana — Failed to retrieve detection engine privileges

**Symptom:**
```
Unable to create actions client because the Encrypted Saved Objects plugin is missing encryption key
```

**Fix:**
```bash
sudo /usr/share/kibana/bin/kibana-encryption-keys generate
```

Add all three output lines to `/etc/kibana/kibana.yml`, then:
```bash
sudo systemctl restart kibana
```

---

### Filebeat events being dropped (count stays 0)

**Symptom:**
```
Failed to index N events in last 10s: events were dropped!
```

**Fix 1 — Delete old index and registry:**
```bash
curl -k -u elastic:YOUR_PASSWORD -X DELETE "https://localhost:9200/siem-events"
sudo systemctl stop filebeat
sudo rm -rf /var/lib/filebeat/registry
sudo systemctl start filebeat
```

**Fix 2 — Regenerate log file:**
The old log file may have wrong schema. Delete it and regenerate:
```bash
rm /home/kali/siem/logs/events_*.jsonl
curl -k -u elastic:YOUR_PASSWORD -X DELETE "https://localhost:9200/siem-events"
sudo systemctl stop filebeat
sudo rm -rf /var/lib/filebeat/registry
python3 log_gen.py   # let it run for 1-2 minutes
sudo systemctl start filebeat
```

---

### Detection rule warning — index not found

**Symptom:**
```
No index matching ["siem_events"] was found
```

**Fix:** The index name uses a hyphen not underscore. Edit the rule and change `siem_events` to `siem-events`.

---

### @timestamp shows ingestion time instead of simulated time

**Symptom:** All events show the same real timestamp instead of simulated 08:00-18:00 times.

**Fix:** Add the timestamp processor to `filebeat.yml`:
```yaml
processors:
  - timestamp:
      field: timestamp
      layouts:
        - '2006-01-02T15:04:05'
      target_field: '@timestamp'
```

Then reset and re-ingest:
```bash
curl -k -u elastic:YOUR_PASSWORD -X DELETE "https://localhost:9200/siem-events"
sudo systemctl stop filebeat
sudo rm -rf /var/lib/filebeat/registry
sudo systemctl start filebeat
```

---

### Generator only shows brute force attacks

**Symptom:** Only `[ATTACK] brute_force` appears, never insider or priv_esc.

**Cause:** The old random attack selection always fell back to brute force when no active sessions existed.

**Fix:** The updated `log_gen.py` uses a round-robin cycle:
```python
attack_cycle = ["brute_force", "insider", "priv_esc"]
```
Download the latest version and replace your file.

---

### Generator shows QUIET spam

**Symptom:**
```
[QUIET ] 18:50 — no users in working hours
[QUIET ] 18:50 — no users in working hours
...
```

**Cause:** The simulated clock reached end of workday (18:30). All users have departed.

**Fix:** Stop the generator with `Ctrl+C` and restart it. The sim clock resets to 08:00 on each run.

---

### Out of memory — VM freezes

**Symptom:** VM becomes unresponsive when starting Elasticsearch + Kibana.

**Fix:**
1. Increase VM RAM to 6GB in VirtualBox/VMware settings
2. Set Elasticsearch heap to 512MB (see heap fix above)
3. Disable ML module: add `xpack.ml.enabled: false` to `elasticsearch.yml`
4. Start services one at a time — Elasticsearch first, wait for it to fully start, then Kibana

---

## Daily Workflow

Once everything is set up, your daily workflow is:

```bash
# Terminal 1 — Start services (if not already running)
sudo systemctl start elasticsearch
sudo systemctl start kibana
sudo systemctl start filebeat

# Terminal 2 — Run generator
cd /home/kali/siem
python3 log_gen.py

# Browser — Monitor alerts
http://localhost:5601 → Security → Alerts
```

---

## Useful Commands

```bash
# Check all services
sudo systemctl status elasticsearch kibana filebeat

# Check event count in Elasticsearch
curl -k -u elastic:YOUR_PASSWORD "https://localhost:9200/siem-events/_count"

# Check recent attack events
curl -k -u elastic:YOUR_PASSWORD "https://localhost:9200/siem-events/_search?q=attack_type:brute_force&pretty&size=3"

# Check Filebeat logs
sudo journalctl -u filebeat -f

# Check Elasticsearch logs
sudo tail -f /var/log/elasticsearch/siem-lab.log

# Reset everything (nuclear option)
curl -k -u elastic:YOUR_PASSWORD -X DELETE "https://localhost:9200/siem-events"
sudo systemctl stop filebeat
sudo rm -rf /var/lib/filebeat/registry
sudo systemctl start filebeat

# Restart all services
sudo systemctl restart elasticsearch kibana filebeat
```

---

## What to Say in a Deloitte Interview

> "I built a single-node SIEM detection lab on Kali Linux simulating a corporate environment with 7 employees across 4 departments. I wrote a Python log generator that produces realistic enterprise telemetry — parallel sessions using Markov-chain user behaviour, meeting and lunch gaps, and three attack types: brute force, insider threat, and privilege escalation. Logs were shipped in real time via Filebeat into Elasticsearch and visualised in Kibana. I wrote five detection rules using KQL and EQL — including a sequence-based EQL rule that detects privilege escalation by correlating sudo execution with user creation within the same session. All rules are mapped to MITRE ATT&CK techniques T1110, T1078, T1041, and T1548."

---

## CV Line

**Detection Engineering Lab — Elastic SIEM**
Built a threat detection pipeline using Filebeat, Elasticsearch 8.x, and Kibana. Modelled insider threat, brute force, and privilege escalation attack chains with ground-truth labels. Wrote KQL and EQL detection rules mapped to MITRE ATT&CK techniques T1110, T1078, T1041, and T1548. Validated real-time alert firing in Elastic SIEM.

---

*Built on Kali Linux — Elastic Stack 8.19 — Python 3.x*
