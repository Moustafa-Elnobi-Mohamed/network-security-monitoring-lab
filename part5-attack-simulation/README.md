# Part 5 — Attack Simulation & Detection

## Objective
Simulate a real-world perimeter network attack using Nmap reconnaissance and Infection Monkey breach simulation, then detect and correlate the attack traffic in Splunk.

---

## 5a — Nmap Perimeter Reconnaissance Scan

### Target
`172.40.0.0/27` — DMZ network segment

### Command Used
```bash
nmap -sS -sU --script auth 172.40.0.0/27
```

**Flag breakdown:**
- `-sS` — TCP SYN (stealth) scan — sends SYN packets, never completes handshake, harder to log
- `-sU` — UDP scan — identifies open UDP services
- `--script auth` — runs Nmap NSE authentication scripts to test for anonymous/default credentials

### Scan Results (172.40.0.30 — TargetLinux01)

| Port | Protocol | State | Service | Finding |
|------|----------|-------|---------|---------|
| 21 | TCP | Open | FTP | ⚠️ Anonymous login allowed (FTP code 230) |
| 22 | TCP | Open | SSH | publickey + password auth — No public keys accepted |
| 23 | TCP | Open | Telnet | 🔴 Plaintext protocol — critical risk |
| 80 | TCP | Open | HTTP | Web server accessible |
| 5900 | TCP | Open | VNC | Remote desktop exposed |
| 6000 | TCP | Open | X11 | X Window System exposed |
| 6667 | TCP | Open | IRC | IRC port open |

**Scan completed:** 32 IP addresses scanned in 95.93 seconds, 2 hosts up

### Risk Assessment

| Service | Risk Level | Reason |
|---------|-----------|--------|
| FTP (Anonymous) | 🔴 Critical | Anyone can authenticate and read/write files |
| Telnet | 🔴 Critical | Credentials and data transmitted in plaintext |
| VNC | 🟠 High | Remote desktop protocol exposed to DMZ |
| X11 | 🟠 High | X Window System allows remote GUI access |
| SSH (password auth) | 🟡 Medium | Susceptible to brute-force attacks |

---

## 5b — Infection Monkey DMZ Breach Simulation

### Setup
- **Launch host:** `clean-ubuntu` (172.31.0.28)
- **Target network:** 172.31.0.0/24
- **Island C2 server:** 172.31.0.28

### Attack Timeline

| Time | Event |
|------|-------|
| 15:17:14 | Monkey starts propagation from clean-ubuntu |
| 15:17:54 | First exploit attempt against 172.31.0.30 using VSFTPDExploiter — **FAILED** |
| 15:17:56 | Second exploit attempt — **SUCCESS** — shell obtained via VSFTPD backdoor |
| 15:21:13 | Monkey finishes execution and reports back |

### Infection Map
The Infection Map showed:
- **clean-ubuntu** (172.31.0.28) as the infection origin
- Successful propagation to **172.31.0.30** via exploit chain
- Scan lines extending to additional hosts in the 172.30.0.0/24 range
- Island C2 at 172.31.0.28 maintaining contact throughout

### Security Report — Critical Findings

#### Immediate Threats
**CVE-2011-2523 — VSFTPD 2.3.4 Backdoor**
- CVSS Score: 10.0 (Critical)
- The VSFTPD 2.3.4 package distributed in 2011 contained an intentionally planted backdoor. Sending a username with `:)` triggers the daemon to open a root shell on port 6200.
- This vulnerability is 15 years old. Its presence in a lab DMZ host demonstrates the risk of running unpatched or legacy services.
- **Remediation:** Remove or upgrade VSFTPD immediately

#### Potential Security Issues
**Weak Network Segmentation**
- Infection Monkey communicated freely between hosts in the 172.30.0.1/24 range
- DMZ hosts should be isolated from each other and from the internal LAN
- **Remediation:** Implement host-based firewall rules + pfSense inter-VLAN blocking

**Weak/Default Passwords**
Monkey tested the following password list and succeeded:
`root`, `1234`, `12345`, `pass`, `user`, `qwe`, `111`, `a1`

---

## 5c — Splunk Detection of Infection Monkey Traffic

### Search Results
Splunk captured 322 events during the attack window. Infection Monkey traffic was correlated across the following Snort signatures:

| Snort Alert | Classification | Source → Destination |
|-------------|---------------|----------------------|
| ET SCAN Potential SSH Scan OUTBOUND | Attempted Information Leak | 172.30.8.1 → 202.28.1.2 |
| GPL WEB_SERVER 403 Forbidden | Attempted Information Leak | 172.30.8.1 → 202.20.1.2:80 |
| GPL WEB_SERVER .htaccess access | Attempted Information Leak | 172.30.8.1 → 172.31.0.38:80 |
| ET SCAN Nmap Scripting Engine User-Agent | Attempted Information Leak | 172.30.8.1 → various |
| ET SCAN Potential SYN Scan OUTBOUND | Attempted Information Leak | 172.30.8.1 → various |

✅ **Splunk successfully correlated attack traffic and made the source IP attributable within the event data.**

### What Splunk Detected Well
- Nmap scan signatures were correctly classified
- SSH scan activity was flagged immediately
- All events were timestamped and searchable

### What Could Be Improved
- Raw syslog format made manual triage slower
- No automated alert fired — a human had to actively search
- No geo-enrichment on external IPs
- See [`/analysis/splunk-improvement-memo.md`](../analysis/splunk-improvement-memo.md) for full recommendations

---

## Screenshots
- `screenshots/nmap-scan-report.png` — Zenmap showing full scan results for 172.40.0.0/27
- `screenshots/infection-monkey-map.png` — Infection Map showing propagation from clean-ubuntu
- `screenshots/infection-monkey-report.png` — Security Report with CVE-2011-2523 and segmentation findings
- `screenshots/splunk-monkey-traffic.png` — Splunk search showing correlated Infection Monkey events
