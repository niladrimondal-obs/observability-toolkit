# observability-toolkit

> AI-powered observability tools built on Dynatrace, Splunk, and OpenTelemetry

Built by an Observability & AIOps Architect with 16 years and 1M+ nodes of real-world scale.  
All tools solve problems I've encountered running platforms for 1,000+ enterprises.

---

## 🗂 Projects in This Repo

### 1. Daily Incident Digest Tool *(Coming Week 3)*
Connects to Dynatrace API → fetches all problems from last 24 hours → formats a clean daily digest.  
**Stack:** Python · Dynatrace REST API · python-dotenv  
**Why it matters:** Teams waste 30–60 mins every morning manually checking dashboards. This eliminates that.

---

### 2. DT → Splunk Bridge *(Coming Week 7)*
Pulls Dynatrace problem data and forwards it to Splunk ITSI for cross-platform correlation.  
**Stack:** Python · Dynatrace REST API · Splunk HEC  
**Why it matters:** Most enterprises run both. Almost nobody has automated the bridge.

---

### 3. DQL Cookbook *(Coming Week 9)*
A curated library of Dynatrace Grail (DQL) queries for real-world observability use cases — infrastructure health, SLO tracking, cost analysis, security signals.  
**Stack:** DQL · Grail · Markdown  
**Why it matters:** DQL is powerful but underdocumented. This becomes a community resource.

---

### 4. Auto-Remediation Workflow *(Coming Week 10)*
Detects a threshold breach in Dynatrace AutomationEngine → creates ServiceNow incident → triggers remediation script → closes ticket automatically.  
**Stack:** Dynatrace AutomationEngine · ServiceNow REST API · Python  
**Why it matters:** This is the full loop: detect → correlate → remediate → verify. No human in the loop.

---

### 5. Multi-Step RCA Agent *(Coming Week 11)*
A LangGraph-based agent that reads Dynatrace problem data and performs multi-step root cause analysis — querying metrics, logs, traces, and topology in sequence.  
**Stack:** Python · LangGraph · Dynatrace API · OpenAI  
**Why it matters:** Single-step AI tools break on complex incidents. This demonstrates production-grade reasoning patterns.

---

### 6. ObservaBot ★ *(Coming Week 14)*
Ask questions about your infrastructure in plain English. Get answers from your Dynatrace environment.  
*"How many critical problems are open in production right now?"*  
*"Which services have had the highest error rate in the last hour?"*  
**Stack:** Python · LangChain · Grail/DQL · Dynatrace REST API  
**Why it matters:** This is the natural language interface for observability. Most teams are 2 years away from needing this. I'm building it now.

---

### 7. Dynatrace MCP Server *(Coming Week 18)*
A Model Context Protocol server that lets any LLM — Claude, GPT-4, Gemini — query your Dynatrace environment directly.  
**Stack:** Python · Anthropic MCP · Dynatrace REST API  
**Why it matters:** MCP is the emerging standard for AI-to-tool communication. This puts Dynatrace on the AI-native infrastructure map.

---

## 🛠 Stack Used Across This Repo

Python · Dynatrace REST API · Grail/DQL · LangChain · LangGraph · Splunk HEC · ServiceNow REST API · OpenTelemetry · AutomationEngine · MCP

---

## 👤 Author

**Niladri Sekhar Mondal** — Observability & AIOps Architect | Platform Leader  
16 years · 1M+ nodes · 1,000+ enterprises · 1.5B+ DDUs  
[LinkedIn](https://linkedin.com/in/niladri-sekhar-mondal-work) · niladrimondal.work@gmail.com
