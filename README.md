# Indy SOC Automation (Wazuh → Shuffle → VirusTotal → TheHive → Email)

End-to-end SOC automation pipeline that detects **Mimikatz** on a Windows endpoint using **Sysmon + Wazuh**, enriches indicators with **VirusTotal**, creates an **alert in TheHive**, and notifies the analyst via **email**.

---

## Architecture Diagram

![SOC Automation Architecture](docs/images/architecture.jpg)

**High-level flow**
1. Windows 10 (Sysmon) generates telemetry → Wazuh Agent forwards events
2. Wazuh Manager detects Mimikatz via custom rule **ID 100002**
3. Wazuh forwards JSON alerts to **Shuffle** via webhook integration
4. Shuffle extracts **SHA256** using regex and enriches it via **VirusTotal v3**
5. Shuffle creates a structured alert in **TheHive** and sends an **Email** notification

---

## Tools Used

- **Wazuh** (Manager + Dashboard)
- **Sysmon** (Windows endpoint telemetry)
- **Shuffle SOAR** (automation workflows)
- **VirusTotal API v3** (hash enrichment)
- **TheHive** (alerting / case management)
- **Vultr** (hosting for Wazuh + TheHive)

---

## Setup

> If this repository is public: **do not upload real IPs, usernames, API keys, or tokens**. Use placeholders.

### 1) Infrastructure (Vultr)

Deploy 2 instances (Ubuntu recommended):
- **Wazuh Server**
- **TheHive Server**

Example endpoints (placeholders):
- Wazuh Dashboard: https://<WAZUH_IP>/
- TheHive: http://<THEHIVE_IP>:9000/

Verify Wazuh is running (on the server):
    sudo systemctl status wazuh-manager

Screenshot:
![Wazuh running](docs/images/wazuh-running.png)

---

### 2) Windows 10 Endpoint (Sysmon + Wazuh Agent)

- Install **Sysmon** (using a Sysmon config)
- Install **Wazuh Agent**
- Confirm the agent is enrolled and telemetry is visible in the Wazuh Dashboard

---

## Detection Rule (Mimikatz) — Rule ID 100002

Create / edit local rules on Wazuh.

Typical path:
- /var/ossec/etc/rules/local_rules.xml

Example rule (adapt fields to match your logs):
    <group name="local,sysmon,windows,">
      <rule id="100002" level="15">
        <if_group>sysmon_event1</if_group>
        <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
        <description>Mimikatz execution detected (Sysmon)</description>
        <mitre>
          <id>T1003</id>
        </mitre>
      </rule>
    </group>

Screenshot:
![Creating custom rule](docs/images/creating-custom-rule.png)

---

## Wazuh → Shuffle Integration (Webhook)

Configure Wazuh to forward alerts to Shuffle when rule **100002** triggers.

Typical path:
- /var/ossec/etc/ossec.conf

Example integration block:
    <integration>
      <name>shuffle</name>
      <hook_url>https://shuffler.io/api/v1/hooks/<WEBHOOK_ID></hook_url>
      <rule_id>100002</rule_id>
      <alert_format>json</alert_format>
    </integration>

Screenshots:
![Integration](docs/images/intergration.png)
![Listening on alerts where rule id is 100002](docs/images/lisening-on-any-alerts-were-rule-id-is-100002.png)

---

## Shuffle Workflow (Webhook → Regex SHA256 → VirusTotal → TheHive → Email)

### Step 1) Webhook Trigger
- Create a Webhook trigger node in Shuffle
- Copy the webhook URL into Wazuh `ossec.conf`

Screenshot:
![Mimikatz detected on Shuffle](docs/images/mimikatz-detected-on-shuffle.png)

---

### Step 2) Extract SHA256 using Regex
Use Shuffle Tools (Regex capture group) to extract the hash.

Regex:
    SHA256=([0-9A-Fa-f]{64})

Screenshot:
![SHA256 performing correctly](docs/images/SHA256-performing-correctly-on-shuffle.png)

---

### Step 3) VirusTotal Enrichment (v3)
Use VirusTotal v3 hash lookup/report using the captured SHA256.

Screenshot:
![VirusTotal performing correctly](docs/images/virus-total-performing-correctly-on-shuffle.png)

---

### Step 4) Create TheHive Alert

Common endpoint:
- POST /api/v1/alert

**Important:** TheHive needs valid JSON and a unique `sourceRef`.

Example JSON body (valid):
    {
      "type": "external",
      "source": "Wazuh",
      "sourceRef": "$exec.id",
      "title": "$exec.title",
      "description": "$exec.text",
      "severity": 3,
      "tags": ["wazuh", "mimikatz", "soc-automation"]
    }

If you hit a severity error, set `severity` based on the alert or use a fixed integer that TheHive accepts.

Screenshots:
![TheHive JSON error](docs/images/thehive-on-shuffler-didnt-execute-properly-cux-of-Json-error.png)
![TheHive successfully working](docs/images/thehive-on-shuffler-sucesfully-working.png)
![Severity fix](docs/images/error-becuase-it-didnt-like-the-exec-sevierty-so-i-change-it-to-the-sevierty-of-the-alert.png)

---

### Step 5) Email Notification
Send an email to the SOC analyst mailbox after TheHive alert creation.

Screenshots:
![Testing email](docs/images/testing-if-the-email-feature-works-on-shuffler.png)
![Successfully received email](docs/images/sucessfully-recived-an-email.png)

---

## Results (Evidence)

- Wazuh detection triggered for Mimikatz (Rule ID 100002)
- Shuffle workflow executed end-to-end (webhook → extraction → enrichment → alerting → notification)
- VirusTotal enrichment completed successfully
- TheHive alert created successfully after resolving JSON/severity issues
- Email notification received by analyst

Screenshots:
![Mimikatz usage detected](docs/images/mimikatz-usage-detected.png)
![Shuffle run - mimikatz detected](docs/images/shuffler-mimikatz-detected.png)

---

## Troubleshooting

### TheHive “Invalid json” error
**Cause:** The request body isn’t valid JSON (common causes: missing commas/quotes, unescaped characters, or injecting variables that break JSON).

**Fix:**
- Use the known-good JSON template above
- Keep `severity` as an integer (e.g., 1–4 depending on TheHive config)
- Avoid placing raw multiline text into JSON without proper escaping (use Shuffle variables that output safe JSON strings)

---

## Next Improvements

- Add automatic observable extraction (IP/domain/hash) and attach observables to TheHive alerts
- Deduplicate repeated alerts (same host/hash within time window)
- Add MITRE tags automatically based on rule mapping
- Add analyst runbook links in TheHive descriptions
- Add response actions (isolate host / disable account) via approved tooling

---

## Screenshots: required folder structure

Put your images here so the README renders them:
- docs/images/

If your filenames differ, update the image paths in this README to match.
