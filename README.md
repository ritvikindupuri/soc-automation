# 🛡️ Enterprise-Grade Splunk & AI Threat Hunting Automation (n8n)

## Executive Summary
This project is an advanced **Security Orchestration, Automation, and Response (SOAR)** pipeline built in **n8n**. It functions as an autonomous, Agentic AI SOC Analyst that continuously monitors Splunk telemetry, investigates suspicious behavior, and executes tiered incident response.

Instead of relying on static, hard-coded rules, this system utilizes an **AI Threat Analyzer (powered by GPT-4)** equipped with custom tools. When a brute-force or anomalous login threshold is breached, the AI autonomously queries threat intelligence, searches historical Splunk data, calculates risk scores, and can even trigger active IP blocking before routing the incident to the appropriate ticketing platform (ServiceNow, PagerDuty, or Jira).

## System Architecture
The workflow is a complex, multi-stage pipeline combining deterministic log parsing with non-deterministic, agentic AI analysis.

<p align="center">
  <img src=".assets/Screenshot 2026-02-06 200956.png" alt="Splunk & AI Threat Hunting Architecture" width="850"/>
  <br>
  <b>Figure 1: Complete Splunk & AI Automation Architecture</b>
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

### 🌊 FLOW 1: Splunk Ingestion & Pre-Processing
This flow handles the high-volume deterministic filtering before engaging the AI, ensuring computational resources are only spent on genuine anomalies.
* **Poll Splunk Every 5 Minutes:** Cron-based schedule trigger ensuring continuous monitoring.
* **Get Security Logs from Splunk:** Executes a predefined search query to pull recent authentication and security events.
* **Split Log Entries & Filter Failed Logins:** Normalizes the JSON arrays and filters the stream specifically for failed authentication attempts.
* **Extract Key Fields & Collect Attempts:** Isolates the source IP, timestamp, and target user.
* **Count Attempts by IP & Check Threshold Exceeded:** A custom code node aggregates attempts per IP over the 5-minute window. An IF node acts as the gatekeeper—if the threshold (e.g., >10 failed logins) is exceeded, the flow proceeds to the AI.

### 🧠 FLOW 2: Agentic AI Threat Analysis
When a threshold is breached, the data is passed to the **AI Threat Analyzer**, an advanced LangChain-based agent node equipped with conversational memory and a suite of active tools.
* **OpenAI GPT-4 (LLM):** The core reasoning engine.
* **Threat Context Memory:** Maintains state during the investigation to correlate multi-step attack patterns.
* **Agent Tools (The AI's Capabilities):**
    * *Threat Intelligence API Tool:* Reaches out to external APIs (e.g., AbuseIPDB, VirusTotal) to check IP reputation.
    * *Risk Score Calculator:* A mathematical tool the AI uses to assign a quantitative risk severity.
    * *Splunk Investigation Tool:* Allows the AI to autonomously query Splunk for *other* activities performed by the suspicious IP.
    * *Threat Intelligence Knowledge Base:* A Pinecone vector store using OpenAI Embeddings to recall historical company threat profiles.
    * *Active Response Tools:* Includes a *Block IP Address* webhook, a *Jira Ticket Tool*, and a *PagerDuty Incident Tool* enabling the AI to take immediate containment actions.

### 📢 FLOW 3: Multi-Channel Alerting
Once the AI reaches a conclusion, it standardizes the output and immediately notifies the team.
* **Format Alert Message:** Parses the AI's reasoning into a readable summary.
* **Send to Splunk:** Writes the AI's findings back into Splunk as a new, enriched security event.
* **Notify Slack Channel & Email Security Team:** Pushes the immediate context to human analysts for situational awareness.

### 🚦 FLOW 4: Severity Routing & Automated Ticketing
The AI's Structured Threat Output is parsed and routed based on the calculated severity level (Critical, High, Medium, Low).
* **Critical Path:**
    * *Prepare Critical Incident Data* ➔ *Create PagerDuty Alert* ➔ *Aggregate Incident Reports* ➔ *Create ServiceNow Incident*.
    * Ensures immediate page-outs for on-call engineers and formal P1 tracking in ServiceNow.
* **High Path:**
    * *Prepare High Priority Ticket* ➔ *Create Jira Security Ticket*.
    * Routes to the standard SOC backlog for investigation.
* **Medium/Low Path:** * Bypasses active paging to prevent alert fatigue, flowing directly into the correlation engine.

### 💾 FLOW 5: Advanced Threat Correlation & Storage
All processed threats, regardless of severity, are cataloged for future intelligence.
* **Advanced Threat Correlation:** Merges the new AI findings with historical data and incoming incident tickets.
* **Store Threat Intelligence:** Pushes the structured JSON into a persistent Data Table or external SQL database.
* **Detect New Threats:** Compares the stored dataset against incoming indicators to identify net-new threat actors (Zero-Days or novel infrastructure).

---

## 🛠️ Configuration & Setup Guide

### Step 1: Splunk Integration
1.  Generate a Splunk **HEC (HTTP Event Collector) Token** or REST API credentials.
2.  In n8n, configure the **Splunk** credentials.
3.  Update the *Get Security Logs from Splunk* node with your specific index and search query (e.g., `index=security sourcetype=linux_secure action=failure`).

### Step 2: Configure OpenAI & Vector DB
1.  **OpenAI:** Add your API key in the n8n credentials. Ensure you have access to `gpt-4` or `gpt-4-turbo`.
2.  **Embeddings/Knowledge Base:** Set up a Pinecone index (1536 dimensions for OpenAI). Connect the *Threat Intelligence Knowledge Base* node using your Pinecone API key and Environment string.

### Step 3: Configure Agent Tools (HTTP Requests)
The AI Agent relies on API connections to take action. Ensure credentials are set for:
* **Threat Intelligence API Tool:** Header Auth for your preferred provider (e.g., AbuseIPDB `Key`).
* **Block IP Address:** Configure the POST request to point to your edge firewall/WAF API (e.g., Palo Alto, Cloudflare).

### Step 4: Ticketing & Notification Integrations
* **Slack:** Use a Bot User OAuth Token (`xoxb-`) with `chat:write` permissions. Set the target channel in the *Notify Slack Channel* node.
* **PagerDuty:** Generate an API Integration Key for your Security service and add it to the *PagerDuty Incident Tool* / *Create PagerDuty Alert* nodes.
* **Jira & ServiceNow:** Add standard Basic Auth or OAuth2 credentials for your respective ITIL platforms to enable automated ticket creation.

---

## 📊 Impact & Outcomes
* **Zero-Touch Triage:** The AI Threat Analyzer autonomously handles the initial 15-20 minutes of investigation (correlating logs, checking threat intel, querying Splunk) instantly.
* **Alert Fatigue Elimination:** By filtering out standard failed logins and only escalating threshold breaches enriched by AI context, SOC analysts only see high-fidelity, actionable alerts.
* **Active Containment:** The agentic architecture allows the system to not just alert, but actively execute IP blocks on the firewall during Critical incidents.
* **Unified Compliance:** Every step—from Splunk ingestion to AI reasoning to ServiceNow ticket creation—is completely logged and auditable.
