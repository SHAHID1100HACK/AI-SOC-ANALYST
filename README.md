# AI-Driven SOC Pipeline: Wazuh + n8n + Gemini 🛡️🤖

## Overview
This project is an automated Level 1 Security Operations Center (SOC) pipeline. It bridges a Wazuh SIEM with a Large Language Model (Google Gemini) using n8n to automatically triage security alerts, enrich them with threat intelligence, log the results, and send real-time alerts to Discord.

This completely replaces the manual triage process for standard alerts, allowing security teams to focus exclusively on AI-verified "True Positives."

## Architecture & Workflow
1. **Wazuh SIEM (Trigger):** Detects an anomaly and triggers a custom integration script to forward the alert payload.
2. **n8n Webhook:** Receives the raw JSON alert.
3. **VirusTotal API (Enrichment):** Extracts the source IP or file hash and queries VirusTotal for community detection data.
4. **Merge Node:** Combines the raw Wazuh log and the VirusTotal threat intelligence into a single JSON payload.
5. **Google Gemini (AI Triage):** Analyzes the combined payload using a strict prompt and outputs a structured technical summary (True Positive, False Positive, or Benign Positive).
6. **If Node (Logic):** Filters the AI's output. Only "True Positives" proceed.
7. **Google Sheets (Logging):** Appends the confirmed threat details to a centralized incident tracker.
8. **Discord Webhook (Alerting):** Pings the security channel with a formatted, high-signal alert.

---

## 🛠️ Step 1: Wazuh Integration Script
Wazuh requires a custom Python script to cleanly format and send alerts to the n8n webhook. 

1. Create the integration script on your Wazuh Manager:
  ```bash
sudo nano /var/ossec/integrations/custom-n8n.
   ```
2. Paste the following Python code (replace the URL with your n8n Production Webhook URL):
```
#!/usr/bin/env python3
import sys
import json
import requests

# Read configuration parameters
alert_file = sys.argv[1]
webhook_url = "YOUR_N8N_WEBHOOK_URL_HERE"

# Read the alert file
with open(alert_file) as f:
    alert_json = json.load(f)

# Send the alert to n8n
headers = {'content-type': 'application/json'}
response = requests.post(webhook_url, json=alert_json, headers=headers)

sys.exit(0)
```

3. Set the correct permissions so Wazuh can execute it:
```
sudo chmod 750 /var/ossec/integrations/custom-n8n.py
sudo chown root:wazuh /var/ossec/integrations/custom-n8n.py
```

## ⚙️ Step 2: Wazuh Manager Configuration

Tell Wazuh to trigger the script whenever an alert hits level 10 or higher.

1. Edit the Wazuh configuration file:

        sudo nano /var/ossec/etc/ossec.conf
2. Add this block inside the <ossec_config> section:
```
  <integration>
    <name>custom-n8n.py</name>
    <level>10</level>
    <hook_url>YOUR_N8N_WEBHOOK_URL_HERE</hook_url>
    <alert_format>json</alert_format>
  </integration>
```

3. Restart the Wazuh Manager to apply changes (using Docker):
```
sudo docker compose restart wazuh.manager
```

## 🧠 Step 3: n8n Node Configurations & Prompts
1. Merge Node

    Mode: Combine

    Combine By: Position

    Output Data From: Both Inputs Merged Together

2. Google Gemini Node Prompt

Use this exact prompt to enforce the AI to act as a Level 1 Analyst:
```

Act as an experienced Level 1 SOC Analyst. You are triaging a security alert from a Wazuh SIEM. Your goal is to provide a clear, technical, and high-signal summary for a human Tier 2 responder.

Alert Context:
    Rule Name: {{ $json["body"]["rule"]["description"] || "N/A" }}
    Raw Log Data: {{ $json["body"]["full_log"] || "N/A" }}
    Agent Host: {{ $json["body"]["agent"]["name"] || "N/A" }}
    VirusTotal Status: {{ $json["data"]["attributes"]["last_analysis_stats"]["malicious"] }} malicious / {{ $json["data"]["attributes"]["last_analysis_stats"]["harmless"] }} harmless

Task:
Analyze the raw log and the VirusTotal reputation for indicators of compromise (IoCs). Provide your response in the following STRICT format:

    VERDICT: [True Positive | False Positive | Benign Positive]
    SEVERITY: [Low | Medium | High | Critical]
    SUMMARY: (1-2 sentences technical description of the activity.)
    REASONING: (Why did you choose this verdict? Mention specific log artifacts and VirusTotal reputation hits.)
    NEXT ACTION: (e.g., "Block Source IP," "Reset User Password," or "No action needed.")

Constraint: Do not be conversational. Just provide the five points above.
```
3. If Node (Filtering logic)

    Value 1: ``` {{ $json.content.parts[0].text }}```

    Operator: Contains

    Value 2:``` VERDICT: True Positive```

   

5. Google Sheets Node (Logging)

Map these expressions to your columns:

  Timestamp: ```{{ $now }}```

  Attacker IP: ```{{ $json.body?.data?.srcip || "Local/Internal" }}```

  Detection: ```{{ $json.body?.rule?.description || "Unknown Alert" }}```

  Gemini Verdict:``` {{ $json.content.parts[0].text }}```

   Status: ```{{ $json.body?.data?.srcip ? "Blocked" : "Manual Review Required" }}```

5. Discord Webhook Node (Alerting)

Paste this into the Discord Message field for clean formatting:
```

🚨 **AI SOC ALERT** 🚨
---
**Detection:** {{ $json["Detection"] || "Manual Test" }}

**AI Analysis:** {{ $json["Gemini Verdict"] }}
---
**System:** {{ $json["Agent Name"] || "No Agent" }} | **IP:** {{ $json["Attacker IP"] || "N/A" }}
**Status:** `{{ $json["Status"] }}`
```

## 🎯 Step 4: Testing the Pipeline

To verify the AI triage is working, simulate an attack on a monitored endpoint (e.g., Metasploitable 2 or an Ubuntu server).

Test 1: Local File Inclusion / Reconnaissance
```

cat /etc/shadow
```
Test 2: Network-Based Attack (Shellshock)
```

curl -H "User-Agent: () { :; }; /bin/eject" http://localhost/
```

### Check your Discord channel and Google Sheet. You should see a highly detailed, AI-generated incident report within seconds!




