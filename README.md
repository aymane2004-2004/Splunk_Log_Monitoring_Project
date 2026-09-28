# SIEM SSH Monitoring with Splunk

A hands-on **SIEM lab** using Splunk Enterprise on Ubuntu to detect SSH brute-force attacks and investigate Linux authentication logs — built the way a SOC analyst would triage a real incident.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Platform](https://img.shields.io/badge/platform-Splunk%20Enterprise%2010.4.3-black)
![OS](https://img.shields.io/badge/OS-Ubuntu%2020.04%2F22.04-orange)

---

## Overview

This project demonstrates an end-to-end SIEM pipeline:

**Log generation → Ingestion → Detection → Visualization → Alerting → Incident Response**

It monitors `/var/log/auth.log` on an Ubuntu VM with Splunk Enterprise, detects SSH brute-force activity, and presents findings on a 14-panel **Incident Responder Dashboard**.

---

## Architecture
```
Ubuntu VM (VirtualBox)
├── OpenSSH Server → produces /var/log/auth.log
└── Splunk Enterprise → ingests → indexes → detects → alerts
└── Web UI :8000 → dashboards & search
```
text

---

## What It Does

| Capability | Description |
|---|---|
| **Ingest** | Continuously monitors `/var/log/auth.log` |
| **Detect** | SPL queries for failed logins, brute force, invalid users |
| **Correlate** | Identifies IPs with failed→successful login (compromise signal) |
| **Classify** | Severity scoring (LOW → CRITICAL) by attempt count |
| **Alert** | Scheduled brute-force alert with throttling |
| **Visualize** | 14-panel SOC dashboard organized by IR workflow |

---

## Dashboard Panels

**Overview** — Total Events · Failed Logins · Successful Logins · Unique Attacking IPs · Compromise Indicator

**Trends** — Failed Logins Over Time · Failed vs Success

**Threat** — Top Attacking IPs · Top Targeted Users · Brute Force by Severity

**Compromise** — ⚠️ Failed → Successful Login (highest-priority panel)

**Post-Exploit** — Root Activity · Sudo Escalation · New Users/Groups

**Forensics** — Attack Timeline · Login by Hour · Outcome Breakdown

---

## Detection Queries

**Brute force:**
```spl
index=main source="*auth.log*" "Failed password"
| rex "from (?<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count as attempts by src_ip
| where attempts > 5
| sort - attempts
Compromise indicator (failed → success):

spl
index=main source="*auth.log*" ("Failed password" OR "Accepted password")
| rex "from (?<src_ip>\d+\.\d+\.\d+\.\d+)"
| eval status=if(match(_raw,"Failed password"),"Failed","Success")
| stats count(eval(status="Failed")) as failures,
        count(eval(status="Success")) as successes by src_ip
| where failures > 3 AND successes > 0
Severity classification:

spl
... | eval severity=case(attempts>50,"CRITICAL", attempts>20,"HIGH",
                         attempts>5,"MEDIUM", true(),"LOW")
Alert
Setting	Value
Name	SSH Brute Force Detection
Type	Scheduled every 5 min
Trigger	Results > 0
Throttle	10 min, grouped by src_ip
Severity	High
Results
A test brute-force attack produced:

src_ip	attempts	severity
192.168.20.128	324	CRITICAL
192.168.20.1	6	MEDIUM
Failed logins spiked to ~320 in one 10-minute window

No successful auth from the attacker IP → attack unsuccessful

No persistence (no new accounts) → no compromise

Quick Start
bash
# 1. Prepare Ubuntu
sudo apt update && sudo apt install -y openssh-server acl

# 2. Install Splunk
sudo dpkg -i splunk-10.4.3-linux-amd64.deb
sudo -u splunk /opt/splunk/bin/splunk start --accept-license

# 3. Grant log read permission
sudo setfacl -m u:splunk:r /var/log/auth.log
sudo setfacl -d -m u:splunk:r /var/log/

# 4. Add monitor (Web UI → Settings → Add Data → Monitor)
#    Path: /var/log/auth.log  |  Index: main  |  Sourcetype: linux_secure

# 5. Generate test attack
for i in {1..20}; do ssh fakeuser@localhost; done

# 6. Search
# index=main source="*auth.log*" "Failed password" | stats count by src_ip
Full instructions: lab_setup.md

Repository Structure
├── README.md                          # Overview
├── lab_setup.md                       # Step-by-step build guide
├── dashboard/
│   └── ssh_soc_dashboard.xml          # Splunk dashboard XML
├── queries/
│   └── detections.spl                 # All SPL detection queries
└── Screenshots                        # photos taken during the investigation
Requirements
Component	: Spec
Hypervisor	: VirtualBox 6.1+ / 7.x
Guest OS	: Ubuntu 20.04 or 22.04 LTS
VM Resources    : 4 GB RAM · 2 vCPU · 40 GB disk
Splunk	        : Enterprise 10.x (Free license — 500 MB/day)
