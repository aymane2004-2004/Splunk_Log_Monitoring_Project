# Lab Setup Guide — SIEM SSH Monitoring with Splunk

**Complete step-by-step instructions to build an SSH security monitoring lab using Splunk Enterprise on an Ubuntu virtual machine.**

This guide assumes no prior Splunk experience. Every command, click, and configuration is documented so the lab can be rebuilt reproducibly.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Virtual Machine Setup](#2-virtual-machine-setup)
3. [Ubuntu Preparation](#3-ubuntu-preparation)
4. [Splunk Enterprise Installation](#4-splunk-enterprise-installation)
5. [Run Splunk as a Non-Root User](#5-run-splunk-as-a-non-root-user)
6. [Enable Boot-Start](#6-enable-boot-start)
7. [Verify Splunk Is Running](#7-verify-splunk-is-running)
8. [Access the Splunk Web UI](#8-access-the-splunk-web-ui)
9. [Ingest `/var/log/auth.log`](#9-ingest-varlogauthlog)
10. [Fix Log Read Permissions](#10-fix-log-read-permissions)
11. [Generate Attack Traffic](#11-generate-attack-traffic)
12. [Run Detection Queries](#12-run-detection-queries)
13. [Create the Brute-Force Alert](#13-create-the-brute-force-alert)
14. [Build the Incident Responder Dashboard](#14-build-the-incident-responder-dashboard)
15. [Troubleshooting](#15-troubleshooting)
16. [Cleanup & Teardown](#16-cleanup--teardown)

---

## 1. Prerequisites

### Hardware

| Resource | Minimum | Recommended |
|---|---|---|
| Host RAM | 4 GB free for VM | 8 GB |
| Host CPU | 2 cores | 4 cores |
| Host Disk | 40 GB free | 60 GB |
| Network | NAT or Bridged | Bridged (for external SSH tests) |

### Software

| Software | Version | Purpose |
|---|---|---|
| Oracle VirtualBox | 6.1+ / 7.x | Hypervisor |
| Ubuntu Server or Desktop | 20.04 LTS or 22.04 LTS | Guest OS |
| Splunk Enterprise | 10.x (Free license) | SIEM platform |
| OpenSSH Server | Any | Generates auth.log |
| A web browser | Any modern | Splunk Web UI |

### Downloads

- **Ubuntu ISO:** https://ubuntu.com/download/server
- **VirtualBox:** https://www.virtualbox.org/wiki/Downloads
- **Splunk Enterprise `.deb`:** https://www.splunk.com/en_us/download/splunk-enterprise.html

> **Note:** Splunk Free license allows up to **500 MB/day** of indexing — more than enough for this lab.

---

## 2. Virtual Machine Setup

### 2.1 Create the VM in VirtualBox

1. Open **VirtualBox** → **New**
2. Configure:

   | Field | Value |
   |---|---|
   | Name | `splunk-siem-lab` |
   | Type | Linux |
   | Version | Ubuntu (64-bit) |
   | Memory | 4096 MB |
   | CPU | 2 cores |
   | Disk | 40 GB (VDI, dynamically allocated) |

3. Click **Create**

### 2.2 Attach the Ubuntu ISO and Install

1. Select the VM → **Settings** → **Storage**
2. Under **Controller: IDE**, click the empty CD icon
3. Click the disk icon → **Choose a disk file** → select the Ubuntu ISO
4. Start the VM and complete the Ubuntu installation
5. When prompted, enable **OpenSSH server** during install (or install it later — see §3)

### 2.3 Network Configuration

- **Settings → Network → Adapter 1**
- Attached to: **NAT** (simplest) or **Bridged Adapter** (if you want to SSH in from other machines)

### 2.4 First Boot

Log in to the VM. Note the IP address:

```bash
hostname -I