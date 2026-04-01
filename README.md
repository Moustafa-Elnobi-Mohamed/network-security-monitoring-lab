# 🔐 Network Security Monitoring & Intrusion Detection Lab

> **Hands-on implementation of enterprise-grade network monitoring using pfSense, Snort IDS, Splunk SIEM, and Kiwi Syslog — culminating in a simulated DMZ breach detected in real time.**

![Security](https://img.shields.io/badge/Domain-Network%20Security-red?style=flat-square)
![Tools](https://img.shields.io/badge/Tools-pfSense%20%7C%20Snort%20%7C%20Splunk%20%7C%20Kiwi-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)
![Lab](https://img.shields.io/badge/Course-Network%20Security%2C%20Firewalls%20%26%20VPNs-orange?style=flat-square)

---

## 📌 Overview

This project documents a full end-to-end network security monitoring lab environment built on virtualized infrastructure. The lab spans firewall log configuration, intrusion detection deployment, centralized log aggregation, and adversary simulation — mirroring a real-world SOC (Security Operations Center) workflow.

**Key outcomes:**
- Deployed and tuned Snort IDS on both LAN and DMZ interfaces
- Centralized logs from pfSense into Splunk Enterprise and Kiwi Syslog Server
- Simulated a real perimeter network attack using Nmap with auth scripts
- Executed a DMZ breach with Infection Monkey, triggering live Snort and Splunk alerts
- Identified a critical **CVE-2011-2523** (VSFTPD backdoor) through the breach simulation

---

## 🗂️ Repository Structure

```
network-security-monitoring-lab/
├── README.md                        ← You are here
├── part1-pfsense-firewall/
│   ├── README.md                    ← pfSense firewall log configuration guide
│   └── screenshots/
│       ├── system-logs.png
│       └── firewall-logs.png
├── part2-snort-ids/
│   ├── README.md                    ← Snort IDS setup and tuning guide
│   └── screenshots/
│       ├── pass-lists-configured.png
│       ├── snort-active-lan.png
│       ├── snort-active-dmz.png
│       ├── ping-results.png
│       ├── icmp-alerts-snort.png
│       └── gplv2-rules-livereload.png
├── part3-kiwi-syslog/
│   ├── README.md                    ← Kiwi Syslog Server forwarding guide
│   └── screenshots/
│       └── kiwi-pfsense-events.png
├── part4-splunk-siem/
│   ├── README.md                    ← Splunk Enterprise indexing and search guide
│   └── screenshots/
│       ├── splunk-indexed-events.png
│       └── splunk-search-results.png
├── part5-attack-simulation/
│   ├── README.md                    ← Attack simulation walkthrough
│   └── screenshots/
│       ├── nmap-scan-report.png
│       ├── infection-monkey-map.png
│       ├── infection-monkey-report.png
│       └── splunk-monkey-traffic.png
└── analysis/
    ├── dmz-breach-summary.md        ← Written analysis of breach findings
    └── splunk-improvement-memo.md   ← Recommendations memo
```

---

## 🛠️ Tools & Technologies

| Tool | Role | Version/Edition |
|------|------|-----------------|
| **pfSense** | Firewall / Gateway | Community Edition |
| **Snort** | Intrusion Detection System (IDS) | GPLv2 Community Rules |
| **Splunk** | SIEM / Log Analysis | Enterprise (Trial) |
| **Kiwi Syslog Server** | Syslog Aggregation | Free Edition |
| **Infection Monkey** | Breach & Attack Simulation | v1.9+ |
| **Nmap / Zenmap** | Network Reconnaissance | Latest |
| **VMware** | Virtualization Platform | Workstation |

---

## 🔬 Lab Sections

### Part 1 — pfSense Firewall Log Configuration
Configured the pfSense firewall to generate and display both **system logs** and **firewall traffic logs**. Observed blocked ICMP traffic originating from the WAN interface (source: `202.20.1.2`, destination: `202.20.1.1`), confirming the firewall ruleset was active and logging correctly.

**Key observations:**
- 64 firewall log entries captured, primarily blocked ICMP and UDP traffic
- System log showed successful admin login, OpenVPN tunnel restarts, and gateway alarms
- Snort startup errors (`/etc/snort/snort_4819_em0`) flagged missing config directory — resolved by completing Part 2 setup

---

### Part 2 — Snort Intrusion Detection System
Deployed Snort as an IDS on both the **LAN (em0)** and **DMZ (em2)** interfaces using the **AC-BNFA** pattern matching engine. Configured a Pass List (`passlist_LAN_IDS`) assigned to the `LAN_HOME_NETWORK_IDS` alias to prevent false positives from internal traffic.

**Detection test:**
Issued ping commands from pfSense's Diagnostics tool targeting `172.30.0.2` via the DMZ source address. Snort successfully logged **6 ICMP alert entries** in the Active Log, confirming detection was operational.

**Alert details:**
```
Proto: ICMP | Class: Misc Activity | Source: 172.31.0.1 → Dest: 172.30.0.2
SID: 2100366 — GPL ICMP PING *NIX
SID: 2100368 — GPL ICMP PING BSDtype
```

**GPLv2 Community Rules** were enabled and confirmed with a live-reload message (`Snort is live-reloading the new ruleset`).

---

### Part 3 — Firewall Log Forwarding with Kiwi Syslog Server
Configured pfSense to forward syslog events to **Kiwi Syslog Server** running on the Windows management workstation. Verified receipt of real-time firewall block events in Kiwi's display panel, with entries showing source hostname `172.30.8.1` and Snort-generated block messages.

---

### Part 4 — Splunk Enterprise SIEM Integration
Installed and configured a **Universal Forwarder** to ship pfSense and Snort logs into Splunk Enterprise. Verified data indexing through Splunk's Search & Reporting interface.

**Indexed data confirmed:** 322 events captured across the monitored time window, with fields including source IP, destination IP, priority, protocol, and Snort classification tags.

---

### Part 5 — Attack Simulation & Detection

#### 5a — Nmap Perimeter Scan
Ran an aggressive Nmap scan against the DMZ network (`172.40.0.0/27`) using:
```bash
nmap -sS -sU --script auth 172.40.0.0/27
```

**Discovered open ports on TargetLinux01 (172.40.0.30):**
| Port | Protocol | Service | Finding |
|------|----------|---------|---------|
| 21 | TCP | FTP | Anonymous login allowed (FTP code 230) |
| 22 | TCP | SSH | publickey + password auth enabled |
| 23 | TCP | Telnet | Open — plaintext protocol risk |
| 80 | TCP | HTTP | Open web server |
| 5900 | TCP | VNC | Remote desktop exposed |
| 6000 | TCP | X11 | X Window System exposed |
| 6667 | TCP | IRC | Open IRC port |

> ⚠️ **Anonymous FTP and open Telnet represent critical misconfigurations in a DMZ host.**

#### 5b — Infection Monkey DMZ Breach Simulation
Launched Infection Monkey from `clean-ubuntu` and allowed it to propagate through the DMZ network. The monkey successfully exploited **172.31.0.30** using the **VSFTPDExploiter**.

**Security Report findings:**

| Severity | Finding |
|----------|---------|
| 🔴 Critical | VSFTPD backdoor vulnerability — **CVE-2011-2523** |
| 🟠 High | Weak network segmentation — unauthorized cross-segment communication in `172.30.0.1/24` |
| 🟡 Medium | Brute-force passwords successfully attempted (root, 1234, pass, user, etc.) |

**Splunk detection:** Infection Monkey traffic was successfully correlated in Splunk, showing `ET SCAN Potential SSH Scan OUTBOUND` and `GPL WEB_SERVER .htaccess access` classifications — confirming SIEM detection was operational.

---

## 📊 Key Findings & Analysis

### DMZ Breach Summary
The Infection Monkey simulation revealed that once an attacker gains an initial foothold, **weak segmentation** allows lateral movement across network segments. The VSFTPD backdoor (CVE-2011-2523) enabled the monkey to gain a shell without credentials — a 2011 vulnerability that should never exist in a production DMZ.

### Splunk Log Analysis — Improvement Recommendations
The raw syslog format from Snort makes triage slower than necessary. Recommended improvements:

**Fields to add as dedicated indexed fields:**
- `src_ip` — enable IP-based filtering and geolocation
- `dest_ip` — track targeted internal hosts
- `dest_port` — identify most-targeted services
- `alert_message` — make classification searchable

**Fields to remove/suppress:**
- `punct`, `eventtype`, `linecount` — redundant metadata that clutters active investigations

**Enhancements:**
- Implement Field Extraction rules to auto-parse Snort alert strings
- Create a **real-time alert** that triggers when >10 SSH scan events occur within 60 seconds
- Build a **Top Targeted Ports** dashboard panel for visual threat triage

---

## 💡 Skills Demonstrated

- ✅ Firewall configuration and log management (pfSense)
- ✅ IDS deployment, rule management, and alert tuning (Snort)
- ✅ Centralized log aggregation (Kiwi Syslog, Splunk Universal Forwarder)
- ✅ SIEM search, indexing, and event correlation (Splunk Enterprise)
- ✅ Network reconnaissance (Nmap, Zenmap, auth scripts)
- ✅ Breach and attack simulation (Infection Monkey)
- ✅ Vulnerability identification (CVE-2011-2523 / VSFTPD)
- ✅ Threat report writing and SOC-style analysis

---

## 📄 Written Deliverables

See the [`/analysis`](./analysis/) folder for:
- **DMZ Breach Summary** — full narrative of what Infection Monkey found and why it matters
- **Splunk Improvement Memo** — professional memo written to IT Manager with actionable recommendations

---

## ⚠️ Disclaimer

This lab was conducted entirely within an **isolated virtual machine environment** for educational purposes as part of a Network Security, Firewalls, and VPNs course. All IP addresses, hosts, and traffic are simulated. No real systems were targeted.

---

## 👤 Author

**Moustafa Mohamed**
elnobimostafa@gmail.com
https://www.linkedin.com/in/moustafa-elnobi-mohamed/
