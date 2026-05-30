# 🛡️ Wazuh + VirusTotal Integration Lab — Windows Endpoint Monitoring

![Wazuh](https://img.shields.io/badge/Wazuh-v4.12.0-blue?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=flat-square&logo=docker)
![VirusTotal](https://img.shields.io/badge/VirusTotal-Integrated-394EFF?style=flat-square)
![n8n](https://img.shields.io/badge/n8n-Automation-EA4B71?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20Ubuntu-lightgrey?style=flat-square)

---

## Why I Built This

I've been working as a SOC analyst and I love automation — so my natural next step was to move toward **SOC engineering**. I wanted to go beyond just reading alerts and actually understand how detection pipelines are *built* from the ground up.

This lab was my first deep dive into Wazuh. I had some Linux and networking basics, but SIEM deployment was completely new territory for me. I set myself a goal: deploy a real monitoring stack, connect a Windows endpoint, and wire up automated threat intelligence — all from scratch.

What I ended up building surprised me. The moment I dropped the EICAR test file into `C:\Windows` and watched **Wazuh flag it in real-time**, then saw **VirusTotal automatically return scan results** for that file — it genuinely clicked for me how a SOC pipeline actually works end to end. Sysmon's visibility into Windows events was equally eye-opening; the level of detail it captures (process creation, file access, network connections) made me understand why it's a staple in enterprise environments.

This repo documents everything I did, so you can replicate it or build on top of it.

---

## 📋 Table of Contents

- [Learning Outcomes](#-learning-outcomes)
- [Architecture Overview](#-architecture-overview)
- [Prerequisites](#-prerequisites)
- [Setup Guide](#-setup-guide)
  - [1. Install Wazuh & n8n in Docker](#1-install-wazuh--n8n-in-docker)
  - [2. Integrate Sysmon & Windows Defender](#2-integrate-sysmon--windows-defender)
  - [3. Configure VirusTotal Integration](#3-configure-virustotal-integration)
  - [4. Add File Integrity Rules](#4-add-file-integrity-rules)
  - [5. n8n Automation with Gemini AI](#5-n8n-automation-with-gemini-ai)
- [Testing & What I Observed](#-testing--what-i-observed)
- [Screenshots](#-screenshots)
- [Tools & Technologies](#-tools--technologies)

---

## 🎯 Learning Outcomes

By completing this lab, you will be able to:

- Deploy **Wazuh SIEM** in a Docker environment
- Integrate a **Windows endpoint** as a monitored agent
- Analyze security logs including **failed logins** and **malicious file detections**
- Connect **VirusTotal API** for automated threat intelligence enrichment
- Build **automated alert workflows** using n8n and Gemini AI

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Ubuntu Host (Docker)                  │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Wazuh      │  │   Wazuh      │  │    n8n       │  │
│  │  Indexer     │  │  Manager     │  │  Automation  │  │
│  └──────────────┘  └──────┬───────┘  └──────┬───────┘  │
│                           │                 │           │
└───────────────────────────┼─────────────────┼───────────┘
                            │                 │
              ┌─────────────┘        ┌────────┘
              │                      │
   ┌──────────▼──────────┐  ┌────────▼───────────┐
   │  Windows Endpoint   │  │   VirusTotal API   │
   │  + Wazuh Agent      │  │   + Gemini AI      │
   │  + Sysmon           │  └────────────────────┘
   └─────────────────────┘
```

---

## ✅ Prerequisites

| Requirement | Details |
|-------------|---------|
| Ubuntu Machine | Fresh install recommended |
| VMware/VirtualBox | For VM setup |
| Windows Machine | As the monitored endpoint |
| VirusTotal Account | Free API key required |
| Docker | Installed via script (see below) |
| Git | `sudo apt install git` |

---

## 🚀 Setup Guide

### 1. Install Wazuh & n8n in Docker

**Configure your VM network** (VMnet1, static IP):

```
IP:       192.168.30.130
Subnet:   255.255.255.0
Gateway:  192.168.30.1
```

**Install dependencies and clone the setup repo:**

```bash
sudo apt install git open-vm-tools-desktop

git clone https://github.com/Atlantium-AI/wazuh-n8n-lab-setup
cd wazuh-n8n-lab-setup

sudo ./install-docker.sh
docker --version  # Verify installation

sudo ./docker-spin-up.sh  # Spins up Indexer + Manager + Dashboard + n8n
docker ps  # Confirm all containers are running
```

**Access the dashboards:**

| Service | URL | Credentials |
|---------|-----|-------------|
| Wazuh Dashboard | `https://localhost` | `admin` / `SecretPassword` |
| n8n | `http://localhost:5678` | Set on first login |

After login, install the Wazuh agent on your Windows machine from the dashboard.

---

### 2. Integrate Sysmon & Windows Defender

**Download and install Sysmon** on your Windows endpoint:

```
https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon
```

**Clone the SwiftOnSecurity config:**

```cmd
git clone https://github.com/SwiftOnSecurity/sysmon-config
```

**Install Sysmon** (Run CMD as Administrator):

```cmd
.\sysmon.exe -i sysmonconfig-export.xml
sysmon -c  # Verify installation
```

**Forward Sysmon logs to Wazuh** — Edit `ossec.conf` at:

```
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Add the following inside the `<ossec_config>` block:

```xml
<localfile>
  <log_format>eventchannel</log_format>
  <location>Microsoft-Windows-Sysmon/Operational</location>
</localfile>

<localfile>
  <location>C:\Windows\System32\LogFiles\Firewall\pfirewall.log</location>
  <log_format>syslog</log_format>
</localfile>
```

**Restart Wazuh service:**

```cmd
net stop WazuhSvc
net start WazuhSvc
```

Verify logs appear in: **Wazuh Dashboard → Agents → [Your Agent] → Logs**

---

### 3. Configure VirusTotal Integration

**Step 1 — Update Windows agent to monitor file changes:**

In `C:\Program Files (x86)\ossec-agent\ossec.conf`, replace:

```xml
<directories recursion_level="0" restrict="regedit.exe$|system.ini$|win.ini$">%WINDIR%</directories>
```

With:

```xml
<scan_on_start>yes</scan_on_start>
<alert_new_files>yes</alert_new_files>
<directories check_all="yes" realtime="yes" report_changes="yes" recursion_level="0">C:\Windows</directories>
<directories check_all="yes" realtime="yes" report_changes="yes">C:\Users\*\Downloads</directories>
```

Also update: `<max_eps>50</max_eps>` → `<max_eps>100</max_eps>`

Restart the service: `Restart-Service WazuhSvc`

**Step 2 — Enable File Integrity in Wazuh Manager:**

```bash
# Extract config from Docker container
docker cp single-node-wazuh.manager-1:/var/ossec/etc/ossec.conf ./ossec_edit.conf

sudo chmod 666 ossec_edit.conf
sudo chown $USER:$USER ossec_edit.conf

nano ./ossec_edit.conf
```

Add the VirusTotal integration block — replace `YOUR_VIRUSTOTAL_API_KEY` with your own key from **VirusTotal → Profile → API Key**:

```xml
<!-- VirusTotal Integration -->
<integration>
  <name>virustotal</name>
  <api_key>YOUR_VIRUSTOTAL_API_KEY</api_key>
  <rule_id>100211</rule_id>
  <alert_format>json</alert_format>
</integration>

<integration>
  <name>virustotal</name>
  <api_key>YOUR_VIRUSTOTAL_API_KEY</api_key>
  <rule_id>100201</rule_id>
  <alert_format>json</alert_format>
</integration>
```

```bash
# Copy config back and restart
docker cp ./ossec_edit.conf single-node-wazuh.manager-1:/var/ossec/etc/ossec.conf
docker restart single-node-wazuh.manager-1
```

---

### 4. Add File Integrity Rules

Navigate to: **Wazuh Dashboard → Server Management → Rules → local_rules.xml**

Add the following rules inside the `</group>` tag:

```xml
<!-- Linux /root Directory Monitoring -->
<rule id="100200" level="7">
  <if_sid>550</if_sid>
  <field name="file">/root</field>
  <description>File modified in /root directory.</description>
</rule>

<rule id="100201" level="7">
  <if_sid>554</if_sid>
  <field name="file">/root</field>
  <description>File added to /root directory.</description>
</rule>

<!-- Windows C:\Windows Directory Monitoring -->
<rule id="100210" level="7">
  <if_sid>550</if_sid>
  <field name="file">C:\\Windows</field>
  <description>File modified in C:\Windows directory.</description>
</rule>

<rule id="100211" level="7">
  <if_sid>554</if_sid>
  <field name="file">C:\\Windows</field>
  <description>File added to C:\Windows directory.</description>
</rule>
```

Click **Save**, then **Restart** from the dashboard.

---

### 5. n8n Automation with Gemini AI

Access n8n at `http://localhost:5678` and create a workflow with the following topology:

**Wazuh Webhook → AI Agent → Gemini → Alert Summary**

In the AI Agent node, under **Options → System Message**, configure your agent prompt to query Wazuh alerts and summarize them using Gemini.

> 💡 **My note:** This was the part I enjoyed most — being able to build the automation layer on top of a detection system is exactly the kind of engineering thinking I wanted to develop. Seeing the AI summarize a real alert felt like a proper SOC workflow.

---

## 🧪 Testing & What I Observed

Download the EICAR test malware file and drop it into `C:\Windows`:

```
https://www.eicar.org/download-anti-malware-testfile/
```

This is a harmless, industry-standard test file recognized by all AV engines as malicious — safe to use in any lab.

**What I saw happen:**

- The moment the file landed in `C:\Windows`, Wazuh's File Integrity Monitoring triggered **rule 100211** almost immediately. The speed genuinely surprised me — I expected some delay, but the realtime monitoring picked it up within seconds.
- Wazuh then automatically sent the file hash to VirusTotal, and the alert came back with the scan results showing the file flagged across multiple engines. Seeing that full loop — file drop → FIM alert → VirusTotal lookup — happen automatically made the whole pipeline feel real.
- On the Sysmon side, I could see exactly *which process* dropped the file, with timestamps and parent process details. That level of visibility made me appreciate why Sysmon is used in enterprise SOC environments. Windows Event Logs alone would never give you that.

---

## 📸 Screenshots

*(See `/screenshots` folder in this repo)*

| # | Screenshot |
|---|------------|
| 1 | Wazuh Dashboard — Active Agents |
| 2 | Sysmon Events in Wazuh Logs |
| 3 | File Integrity Alert — EICAR Test File |
| 4 | VirusTotal Alert in Wazuh |
| 5 | n8n Workflow — Automation Canvas |
| 6 | Gemini AI Alert Summary Output |

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| [Wazuh v4.12.0](https://wazuh.com) | SIEM & XDR Platform |
| [Docker](https://docker.com) | Container orchestration |
| [Sysmon](https://docs.microsoft.com/sysinternals) | Advanced Windows logging |
| [VirusTotal](https://virustotal.com) | Threat intelligence API |
| [n8n](https://n8n.io) | Workflow automation |
| [Gemini AI](https://ai.google.dev) | AI-powered alert analysis |

---

## 📄 License

This project is for educational purposes. Feel free to fork, use and build on it.
