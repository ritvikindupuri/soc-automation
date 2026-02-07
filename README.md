# 🛡️ Enterprise-Grade Threat Intelligence Automation in n8n

## Executive Summary
This project is a **38-node security automation workflow** built in **n8n** that functions as an autonomous SOC Analyst. It orchestrates the entire threat lifecycle—from multi-source data collection and deduplication to real-time risk scoring and automated remediation.

The system processes security threats in real-time, leveraging a custom weighted risk engine to distinguish between noise and critical indicators, triggering firewall blocks and P1 tickets only when necessary.

## System Architecture
The workflow is divided into **9 distinct flows** that handle specific stages of the intelligence pipeline.

<p align="center">
  <img src=".assets/Screenshot 2026-02-06 200956.png" alt="Full 38-Node n8n Workflow" width="850"/>
  <br>
  <b>Figure 1: Complete Threat Intelligence Automation Architecture</b>
</p>

## 🚀 Usage: How to Import
To use this automation in your own n8n instance:
1.  Download the `.json` workflow file included in this repository.
2.  Open your n8n workspace.
3.  Click the **three dots (•••)** in the top right corner of the canvas.
4.  Select **"Import from File"** and upload the JSON file.
5.  Follow the *Configuration & Setup Guide* below to configure your credentials.

---

## 🎯 Detailed Workflow Breakdown

### 🌊 FLOW 1: Scheduled Threat Intelligence Collection
* **Node 1: Schedule Trigger**
    * *Purpose:* Initiates the collection pipeline.
    * *Configuration:* Runs at minute 0 of every hour (`0 * * * *`).
* **Node 2: Fetch Threat Feed A (HTTP Request)**
    * *Method:* GET request to primary threat intelligence API.
    * *Headers:* Authenticated via API Key.
    * *Output:* JSON array of raw indicators (IPs, domains, hashes).
* **Node 3: Parse Threat Feed A (Code Node)**
    * *Logic:* Normalizes the API response; extracts IP addresses, threat types, and confidence scores; converts timestamps to ISO 8601.
* **Node 4: Fetch Threat Feed B (HTTP Request)**
    * *Purpose:* Secondary source for correlation.
    * *Method:* GET request with distinct API credentials.
* **Node 5: Parse Threat Feed B (Code Node)**
    * *Logic:* Maps unique fields from Feed B to the standard schema used in Node 3.
* **Node 6: Merge Threat Feeds**
    * *Mode:* Append (keeps all records from both feeds).
* **Node 7: Remove Duplicates (Code Node)**
    * *Logic:* Uses a `Set` data structure to track unique IPs.
    * *Priority:* Retains the entry with the highest confidence score if duplicates are found.

### ⚡ FLOW 2: Real-Time Threat Alert Processing
* **Node 8: Webhook Trigger**
    * *Method:* POST
    * *Path:* `/threat-alert`
    * *Auth:* Header Auth (`X-API-Key`)
    * *Payload:* `{ "ip": "x.x.x.x", "type": "malware", "severity": "high" }`
* **Node 9: Validate Webhook Data (IF Node)**
    * *Conditions:* Checks if IP is present, Severity is valid (low/medium/high/critical), and Timestamp exists.
* **Node 10: Enrich IP Data (HTTP Request)**
    * *API:* IP Geolocation Service.
    * *Data:* Retrieves Country, ISP, and known malicious status.
* **Node 11: Format Enriched Data (Code Node)**
    * *Logic:* Merges geolocation context into the threat object and adds a processing timestamp.

### 🔍 FLOW 3: Manual Threat Investigation (Ad-Hoc)
* **Node 12: Manual Trigger**
    * *Use Case:* Analyst clicks "Test Workflow" for specific threat hunting.
* **Node 13: Input IP Address (Set Node)**
    * *Config:* Defines the specific IP to investigate.
* **Node 14: Check Multiple Databases (HTTP Request - Parallel)**
    * *Simultaneous Queries:* AbuseIPDB, VirusTotal, Shodan, AlienVault OTX.
* **Nodes 15-18: Parse Database Responses (Code Nodes)**
    * *Node 15:* Extracts AbuseIPDB confidence score.
    * *Node 16:* Extracts VirusTotal detection ratio.
    * *Node 17:* Extracts Shodan open ports/services.
    * *Node 18:* Extracts AlienVault pulse associations.
* **Node 19: Aggregate Intelligence (Code Node)**
    * *Logic:* Calculates average confidence, lists all detected threat types, and identifies the highest severity rating across all sources.

### 🧠 FLOW 4: Risk Scoring Engine
* **Node 20: Calculate Base Risk Score (Code Node)**
    * *Formula:*
      ```javascript
      // Weights: Confidence (40%), Detection Count (30%), Recency (20%), Geo (10%)
      const riskScore = (dbCount * 30) + (avgConfidence * 40) + (recencyScore * 20) + (geoRisk * 10);
      ```
* **Node 21: Determine Threat Level (IF Node)**
    * *Critical:* Score 76-100
    * *High:* Score 51-75
    * *Medium:* Score 26-50
    * *Low:* Score 0-25

### 💾 FLOW 5: Data Preparation & Storage
* **Node 22: Prepare Threat Data (Code Node)**
    * *Schema:* `ipAddress`, `threatLevel`, `riskScore`, `timestamp`, `indicators` (JSON string).
* **Node 23: Store Threat Intelligence (Data Table)**
    * *Operation:* Upsert (Update if IP exists, Insert if new).
    * *Match Column:* `ipAddress`.

### 🚨 FLOW 6: Critical Threat Response (>75)
* **Node 24: Format Critical Alert**
    * *Content:* Detailed IP/Geo info, risk breakdown, and recommended actions.
* **Node 25: Send Email Alert**
    * *Subject:* `🚨 CRITICAL THREAT DETECTED: [IP Address]`
    * *Priority:* High.
* **Node 26: Post to Slack**
    * *Channel:* `#security-alerts`
    * *Mentions:* `@security-team`
* **Node 27: Create Incident Ticket**
    * *Integration:* ServiceNow/Jira/PagerDuty.
    * *Priority:* P1 (Critical).
* **Node 28: Trigger Firewall Block (HTTP Request)**
    * *Action:* API call to Firewall (e.g., pfSense/Cisco) to add IP to blocklist permanently.

### ⚠️ FLOW 7: High Threat Response (51-75)
* **Nodes 29-31:** Formats alert, sends non-urgent email, and posts to `#security-monitoring` Slack channel.
* **Node 32: Add to Watch List (HTTP Request)**
    * *Action:* Tags IP for enhanced logging in the monitoring system.

### 📝 FLOW 8: Medium/Low Threat Logging (<51)
* **Nodes 33-35:** Formats log entry, sends payload to SIEM (Splunk/ELK/QRadar), and updates internal threat database.

### 📊 FLOW 9: Reporting & Analytics
* **Node 36: Daily Summary Trigger:** Runs daily at 8:00 AM.
* **Node 37: Query Threat Data:** Retrieves all records where `timestamp > (now - 24h)`.
* **Node 38: Generate Summary Report:** Calculates total threats, severity breakdown, top 10 malicious IPs, and geographic trends.

---

## 🛠️ Configuration & Setup Guide

### Step 1: Create Data Tables (CRITICAL)
*Data tables are not created automatically. You must configure this manually first.*
1.  Go to **Data Tables** tab in n8n.
2.  Create Table Name: `Store Threat Intelligence`
3.  Add Columns:
    * `ipAddress` (Type: Text)
    * `threatLevel` (Type: Text)
    * `riskScore` (Type: Number)
    * `timestamp` (Type: Text)
    * `indicators` (Type: Text)

### Step 2: Configure API Credentials
* **AbuseIPDB:** Create "Header Auth" → Name: `Key`, Value: `[Your_API_Key]`.
* **VirusTotal:** Create "Header Auth" → Name: `x-apikey`, Value: `[Your_API_Key]`.
* **Shodan:** Use "Generic Credential Type" → API Key.

### Step 3: Configure Notifications
* **Email (Gmail):** Use `smtp.gmail.com`, Port `587`, TLS. Use an **App Password** (not your login password).
* **Slack:** Create an App at `api.slack.com`. Add scopes: `chat:write`, `chat:write.public`, `channels:read`. Use the **Bot User OAuth Token** (`xoxb-...`).

### Step 4: Configure Webhook Source
External systems must send POST requests to your production Webhook URL with the header `X-API-Key: [Your_Secret_String]`.
**Expected Payload:**
```json
{
  "ip": "192.168.1.100",
  "type": "malware",
  "severity": "high",
  "timestamp": "2024-01-15T10:30:00Z"
}

### Step 5: SIEM & Firewall Integration
* **Splunk:** POST to `https://splunk:8088/services/collector` with Header `Authorization: Splunk [HEC_TOKEN]`.
* **Firewall (pfSense Example):** POST to `/api/v1/firewall/alias/entry` with payload `{"address": "{{$json.ipAddress}}"}`.

---

## 📊 Impact & Outcomes
* **Response Time:** Critical threats are blocked at the firewall level within seconds of detection.
* **Accuracy:** Multi-source correlation and the custom risk scoring engine drastically reduce false positives.
* **Scalability:** The modular design allows for adding new threat feeds (Nodes 4-5) without disrupting the core logic.
* **Compliance:** Provides a complete, persistent audit trail of all detected threats and automated actions.
