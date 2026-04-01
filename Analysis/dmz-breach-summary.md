# DMZ Breach Simulation — Analysis Report

**Date:** March 29, 2026
**Analyst:** Moustafa Mohamed
**Tool Used:** Infection Monkey v1.9+
**Environment:** Isolated VMware lab network

---

## Executive Summary

A simulated DMZ breach was conducted using Infection Monkey, launched from a clean Ubuntu host within the lab environment. The simulation revealed two critical security failures: an unpatched backdoor vulnerability in an FTP service, and inadequate network segmentation allowing unauthorized cross-segment communication.

---

## Attack Path

1. Infection Monkey was initialized on `clean-ubuntu`
2. It began propagating by scanning the `172.30.0.1/24` subnet
3. Initial exploitation attempt against `172.31.0.30` using VSFTPDExploiter **failed**
4. Second attempt **succeeded** — Monkey gained shell access via the VSFTPD backdoor
5. Monkey finished execution and reported back to the Island C2 server

---

## Vulnerabilities Identified

### CVE-2011-2523 — VSFTPD 2.3.4 Backdoor
- **Severity:** Critical (CVSS 10.0)
- **Description:** A backdoor was intentionally introduced into VSFTPD version 2.3.4. When a username containing `:)` is sent, the daemon opens a shell on port 6200.
- **Impact:** Unauthenticated remote root shell access
- **Remediation:** Upgrade VSFTPD immediately. This vulnerability is over a decade old and has no place in any environment.

### Weak Network Segmentation
- **Severity:** High
- **Description:** The Infection Monkey was able to communicate freely across segments within the `172.30.0.1/24` range. DMZ hosts should never be able to initiate connections to internal LAN segments.
- **Impact:** Lateral movement enabled — an attacker who compromises one DMZ host can pivot inward
- **Remediation:** Implement strict inter-VLAN firewall rules. DMZ to LAN traffic should be blocked by default, with only specific, necessary ports explicitly allowed.

---

## Greatest Concerns from a Network Monitoring Perspective

### 1. Time-to-Detection Gap
The VSFTPD exploit completed before any manual alert triage could occur. Without automated alerting rules in Splunk (e.g., threshold-based triggers on scan volume), a real SOC analyst would likely see the alert *after* the breach was already established.

### 2. Raw Log Format Limits Speed
Snort logs forwarded to Splunk arrived in raw syslog format. Key fields — source IP, destination IP, port, and alert classification — were embedded in long text strings rather than parsed into discrete, searchable fields. This slows investigation significantly during an active incident.

### 3. No Lateral Movement Alerting
The segmentation failure allowed cross-segment traffic, but there were no specific rules in place to alert on *unexpected* east-west traffic patterns. A proper monitoring setup should baseline normal traffic and alert on deviations.

### 4. Brute-Force Passwords Succeeded
The monkey successfully attempted simple passwords (root, 1234, pass, user). This indicates either SSH is exposed with password authentication enabled, or services are running with default credentials — both are serious operational failures.

---

## Recommendations

| Priority | Action |
|----------|--------|
| Immediate | Patch or remove VSFTPD 2.3.4 — replace with a current, maintained FTP solution or eliminate FTP entirely |
| Immediate | Implement pfSense firewall rules blocking all DMZ-to-LAN initiated connections |
| Short-term | Disable password authentication on SSH; enforce public key only |
| Short-term | Parse Snort logs with Splunk Field Extraction for faster triage |
| Medium-term | Build Splunk dashboards for Top Source IPs, Top Targeted Ports, and Alert Classifications |
| Medium-term | Set up automated alert: >10 scan-type events within 60 seconds triggers SOC notification |

---

*This report was produced as part of a Network Security, Firewalls, and VPNs lab exercise in an isolated virtual environment.*
