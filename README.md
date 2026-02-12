# Indy SOC Automation Project

SOC automation pipeline using **Wazuh** (SIEM/XDR) + **Shuffle** (SOAR) + **VirusTotal** (enrichment) + **TheHive** (case management).

## What it does
1. Windows 10 endpoint generates telemetry via Sysmon.
2. Wazuh detects suspicious activity (custom rule: Mimikatz).
3. Wazuh forwards alerts matching `rule_id=100002` to Shuffle via webhook integration.
4. Shuffle:
   - extracts SHA256 from the alert payload
   - enriches it using VirusTotal v3
   - sends an email notification
   - creates an alert in TheHive

## Architecture
![Architecture](docs/images/architecture.jpg)

## Tech
- Wazuh Manager/Indexer/Dashboard (cloud VPS)
- TheHive (cloud VPS)
- Shuffle (cloud)
- VirusTotal API v3
- Windows 10 + Sysmon + Wazuh Agent

## Repository layout (suggested)
```
.
├── docs/
│   ├── Indy_SOC_Automation_Project_Final_Report.docx
│   └── images/
├── configs/
│   ├── wazuh/
│   │   ├── ossec.conf.snippet.xml
│   │   └── local_rules.xml
│   └── shuffle/
│       └── thehive_alert_body.json
└── .github/
```

## Key configs
### Wazuh integration (ossec.conf)
See: `configs/wazuh/ossec.conf.snippet.xml`

### Custom rule (local_rules.xml)
See: `configs/wazuh/local_rules.xml`

### TheHive alert JSON (Shuffle body)
See: `configs/shuffle/thehive_alert_body.json`

## Setup checklist
- [ ] Install Sysmon + Wazuh Agent on the Windows endpoint
- [ ] Deploy Wazuh on a VPS and confirm the dashboard loads (TCP/443)
- [ ] Add the local rule for Mimikatz detection (rule_id `100002`)
- [ ] Configure Wazuh integration hook to Shuffle webhook URL
- [ ] Build Shuffle workflow (Webhook → SHA256 Extract → VirusTotal → Email → TheHive)
- [ ] Create TheHive users: SOC analyst + Shuffle service account
- [ ] Store secrets in `.env` (never commit)

## Environment variables
Copy `.env.example` → `.env` and fill:
- `THEHIVE_URL`, `THEHIVE_API_KEY`
- `VT_API_KEY`
- `EMAIL_TO` (for notifications)

## Notes / troubleshooting
- TheHive rejects duplicate alerts when `sourceRef` is the same. Use a unique `sourceRef`, e.g.:
  `{{ exec.rule_id }}-{{ shuffle.execution_id }}`

## License
MIT
