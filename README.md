<div align="center">

# 🛡️ ALI — Autonomous Log Intelligence

**A self-hosted, AI-powered Security Operations Centre that reads your logs, detects threats, and alerts your team in plain English.**

[![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![n8n](https://img.shields.io/badge/n8n-EA4B71?logo=n8n&logoColor=white)](https://n8n.io/)
[![Ollama](https://img.shields.io/badge/Ollama-000000?logo=ollama&logoColor=white)](https://ollama.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

</div>

---

## Overview

ALI (Autonomous Log Intelligence) is an automated Security Operations Centre powered by a **locally-run AI model**. It continuously collects logs from a monitored system, classifies each event by attack type, uses a local LLM to analyse and rate the threat, then delivers a clear, actionable alert to Slack and email — and saves an incident report.

No cloud. No per-query fees. No SIEM licence. It runs entirely on your own hardware.

> **ALI is a first-pass triage tool, not a replacement for a security analyst.** It does the tireless 24/7 log-reading so your team can focus on investigation and response.

---

## ✨ Features

- 🔄 **Automated collection** — pulls auth, web, and system logs etc on a schedule
- 🧠 **Local AI analysis** — uses Ollama + Llama; no data leaves your network
- 🏷️ **Attack classification** — sorts events into SSH brute force, web attack, privilege escalation, and more
- 🎯 **One alert per attack type** — focused, readable alerts instead of noise
- 🌈 **Plain-English reports** — severity, what happened, and recommended actions
- 📢 **Multi-channel alerting** — Slack (with `@channel` ping) and styled HTML email
- 💾 **Incident archiving** — every analysis saved as a report file
- 🐳 **Fully containerised** — one command to start the whole stack

---

## 🏗️ Architecture

```
┌─────────────────┐         ┌──────────────────────────────────────┐
│  Monitored host │         │              ALI host                │
│                 │         │           (Docker stack)             │
│                 │         │                                      │
│   System logs   │ Filebeat│  ┌──────────┐   ┌──────────┐         │
│   auth / nginx /│────────▶│  │ Logstash │──▶│fileserver│         │
│   syslog        │  :5044  │  │ (parse + │   │  (HTTP)  │         │
│                 │         │  │  tag)    │   └────┬─────┘         │
│                 │         │  └──────────┘        │               │
└─────────────────┘         │                      ▼               │
                            │  ┌──────────┐   ┌──────────┐         │
                            │  │  Ollama  │◀──│   n8n    │         │
                            │  │ (local   │   │(workflow)│──▶ Slack │
                            │  │  LLM)    │   │          │──▶ Email │
                            │  └──────────┘   └──────────┘──▶ File  │
                            └──────────────────────────────────────┘
```

**Data flow:** Filebeat ships logs → Logstash parses and tags them → writes files → the fileserver exposes them over HTTP → n8n pulls them on a schedule → the local LLM analyses → alerts go out.

> **Why a fileserver?** Logstash *pushes* data; n8n *pulls* it. The fileserver bridges the two and decouples them, so if n8n is down, logs still accumulate safely.

---

## 📡 Source Setup — Shipping Logs with Filebeat

The monitored host uses **Filebeat** to ship its logs to the ALI Logstash instance. Run these steps on the machine you want to monitor.

### 1. Install Filebeat

**Debian / Ubuntu:**

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.0-amd64.deb
sudo dpkg -i filebeat-8.12.0-amd64.deb
```

**RHEL / CentOS:**

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.0-x86_64.rpm
sudo rpm -vi filebeat-8.12.0-x86_64.rpm
```

> Keep Filebeat on the same major version as Logstash (this project uses `8.12.0`).

### 2. Configure Filebeat

Edit the config file:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Replace its contents with the following. This collects auth, syslog, and nginx logs, tags each with a `log_type`, and sends them to Logstash:

```yaml
filebeat.inputs:
  - type: filestream
    id: auth-logs
    enabled: true
    paths:
      - /var/log/auth.log
    fields:
      log_type: auth
    fields_under_root: true

  - type: filestream
    id: syslog
    enabled: true
    paths:
      - /var/log/syslog
    fields:
      log_type: syslog
    fields_under_root: true

  - type: filestream
    id: nginx-access
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      log_type: nginx_access
    fields_under_root: true

# Send logs to the ALI Logstash instance
output.logstash:
  hosts: ["ALI_HOST_IP:5044"]

# Disable the default Elasticsearch output
output.elasticsearch:
  enabled: false
```

> Replace `ALI_HOST_IP` with the IP address of your ALI (Logstash) host.

### 3. Test the configuration

```bash
sudo filebeat test config
sudo filebeat test output
```

`test output` should report a successful connection to Logstash on port `5044`.

### 4. Start Filebeat

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
sudo systemctl status filebeat
```

### 5. Verify logs are flowing

On the ALI host:

```bash
docker logs logstash --tail 20
ls -l logstash/events/
```

You should see log files appearing, e.g. `auth-YYYY-MM-DD.log` and `nginx_access-YYYY-MM-DD.log`.

---

## 🚀 ALI Host Setup

### Prerequisites

- Docker & Docker Compose
- At least 8 GB RAM (16 GB recommended for larger models)
- A monitored host running Filebeat (see section above)

### 1. Clone the repository

```bash
git clone https://github.com/softlab-sec/Autonomous-Log-Intelligence.git
cd ali-lab
```

### 2. Start the stack

```bash
docker compose up -d
```

This launches n8n, Ollama, Logstash, and the fileserver.

### 3. Pull the AI model

```bash
docker exec ollama ollama pull llama3.1:8b
# Lighter, faster option:
docker exec ollama ollama pull llama3.2:latest
```

### 4. Warm the model

```bash
docker exec ollama ollama run llama3.1:8b "warmup"
docker exec ollama ollama ps   # confirm the model is loaded
```

### 5. Import the workflow

- Open n8n at `http://localhost:5678`
- Import `workflow/ali-workflow.json`
- Configure your **Slack webhook** and **SMTP credentials**
- Activate the workflow

---

## ⚙️ Configuration

### Key settings

| Setting | Where | Purpose |
|--------|-------|---------|
| Schedule interval | Schedule Trigger node | How often ALI runs (e.g. every 3–5 min) |
| `LINES_PER_SOURCE` | Parse Log node | How many recent log lines to analyse |
| `model` | Build Prompt node | Which Ollama model to use |
| `num_predict` | Build Prompt node | Max length of the AI response |
| Slack webhook | Slack node | Your incoming webhook URL |
| SMTP credentials | Send Email node | Your mail server details |

### Critical Ollama settings (in `docker-compose.yml`)

```yaml
environment:
  - OLLAMA_KEEP_ALIVE=-1        # keep the model warm (avoids slow cold-starts)
  - OLLAMA_MAX_LOADED_MODELS=1
  - OLLAMA_NUM_PARALLEL=1
```

> ⚠️ **Important:** the model in your Build Prompt node **must match** the model warmed in Ollama (`ollama ps`), or every call will trigger a slow cold reload.

---

## 🧩 How the workflow works

| Node | Role |
|------|------|
| **Schedule Trigger** | Fires the workflow on a timer |
| **Build Filename** | Builds today's log file URLs |
| **Fetch Log** | Downloads the log files over HTTP |
| **Parse Log** | Cleans logs and classifies each line by attack type |
| **Has Data?** | Stops the run if there's nothing to report |
| **Build Prompt** | Writes a focused AI prompt per attack type |
| **LLM** | Sends the prompt to the local Ollama model |
| **Parse AI Response** | Extracts fields, builds the formatted alert |
| **Slack / Email** | Delivers the alert |
| **Convert to File → Disk** | Saves the incident report |

---

## 🐛 Troubleshooting

**Filebeat won't connect to Logstash**
Check the ALI host IP and that port `5044` is reachable. Run `sudo filebeat test output`. Ensure the Logstash container is up (`docker ps`).

**LLM node times out**
The model is likely cold or mismatched. Confirm `docker exec ollama ollama ps` shows the same model your Build Prompt requests, and that `OLLAMA_KEEP_ALIVE=-1` is set.

**Fetch Log times out on some files**
Ensure the fileserver uses `ThreadingTCPServer` (not `TCPServer`) so it can serve parallel requests.

**Alerts show empty fields / "undefined"**
The LLM returned nothing. Check the LLM node's output — usually a cold model or an empty-log run.

**AI invents fake threats**
Ensure the empty-log guard is in Build Prompt and the "report only what's in the logs" rule is in the prompt.

**"path is a directory" on file save**
The incidents folder must exist: `docker exec -it n8n mkdir -p /home/node/.n8n-files/incidents`

---

## 🛣️ Roadmap

- [ ] GPU support for faster inference
- [ ] Elasticsearch / Kafka log ingestion
- [ ] SIEM integration
- [ ] Additional attack categories (malware, exfiltration, persistence)
- [ ] Web dashboard for incident history
- [ ] Multi-host monitoring

---

## ⚠️ Disclaimer

ALI is a **proof-of-concept lab project** for learning and demonstration. It is **not** a production-hardened security product. It performs first-pass triage to assist human analysts — it does not replace professional security monitoring, and it does not auto-remediate. Validate all findings independently.

---

## 📄 License

MIT — see [LICENSE](LICENSE).

---

## 🙌 Built with

[Docker](https://www.docker.com/) · [Filebeat](https://www.elastic.co/beats/filebeat) · [Logstash](https://www.elastic.co/logstash) · [n8n](https://n8n.io/) · [Ollama](https://ollama.com/) · [Llama](https://ai.meta.com/llama/)

<div align="center">

**If ALI helped you, drop a ⭐**

</div>
