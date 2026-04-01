# Internal Memo — Splunk SIEM Evaluation & Improvement Recommendations

**TO:** IT Manager
**FROM:** Moustafa Mohamed, Security Analyst
**DATE:** March 29, 2026
**RE:** DMZ Breach Detection via Splunk — Findings and Recommendations

---

## Detection Summary

Splunk Enterprise performed well during our simulated DMZ breach exercise using Infection Monkey. The SIEM successfully ingested Snort IDS alerts forwarded from pfSense and indexed 322 events within the monitored time window. Splunk correctly flagged the following threat signatures in near-real-time:

- `ET SCAN Potential SSH Scan OUTBOUND` — correctly identified reconnaissance activity
- `GPL WEB_SERVER .htaccess access` — flagged unauthorized web server probing
- `ET SCAN Potential SYN Scan OUTBOUND` — detected the Nmap stealth scan

The attacker's source IP (`172.30.8.1`) was identifiable within the event stream, and the timeline of the attack was reconstructable from log entries.

**Verdict: Splunk is a capable and effective SIEM for this environment. The issues below are configuration gaps, not platform limitations.**

---

## Current Issues

### Issue 1 — Raw Syslog Format
Snort alerts are arriving in Splunk as unstructured syslog strings. Fields like Source IP, Destination IP, destination port, and the alert classification (e.g., "Attempted Information Leak") are buried inside a long text field rather than being parsed into individual, indexed columns.

**Impact:** During an active incident, analysts must manually read through raw text rather than filtering or pivoting on clean data fields. This slows triage significantly.

### Issue 2 — Redundant Metadata Fields
Fields such as `punct`, `eventtype`, and `linecount` are being indexed and displayed but carry no investigative value. They add visual clutter to search results and consume index storage unnecessarily.

### Issue 3 — No Automated Alerting
Currently, detection is entirely passive — an analyst must be actively watching the Splunk dashboard to notice an attack. There are no threshold-based alerts configured to notify the team automatically.

---

## Recommendations

### 1. Implement Field Extraction for Snort Logs (Priority: High)
Use Splunk's Field Extraction feature (`Settings > Fields > Field Extractions`) to create a regex-based extraction rule for the Snort syslog format. Target fields to extract:

```
src_ip       — Source IP address of the alert
dest_ip      — Destination IP address
dest_port    — Targeted port number
priority     — Snort priority level (1, 2, or 3)
proto        — Protocol (TCP, UDP, ICMP)
alert_msg    — The full Snort rule description
classification — The threat category (e.g., Attempted Information Leak)
```

Once extracted, these fields become filterable columns, enabling pivot tables, IP-based lookups, and chart generation.

### 2. Suppress Redundant Fields (Priority: Medium)
Add the following to `transforms.conf` or use Splunk's field aliasing to suppress display of fields with no investigative value:
- `punct`
- `eventtype`
- `linecount`
- `splunk_server_group`

### 3. Configure Automated Alerts (Priority: High)
Create the following Splunk saved searches with alert actions:

**Alert A — SSH Scan Surge**
```
search index=snort classification="Potential SSH Scan" | stats count by src_ip | where count > 10
```
Trigger: Every 1 minute. Action: Send email or Slack notification to SOC.

**Alert B — High Priority Snort Event**
```
search index=snort priority=1
```
Trigger: Real-time. Action: Immediate notification.

### 4. Build a SOC Dashboard (Priority: Medium)
Create a Splunk dashboard with the following panels:
- **Top 10 Source IPs** — who is generating the most alerts
- **Top Targeted Ports** — which services attackers are hitting most
- **Alert Classifications Over Time** — trend view for escalation patterns
- **Alert Volume by Hour** — identify attack windows and off-hours activity

---

## Conclusion

Splunk successfully detected traces of the simulated Infection Monkey breach in real time. With the field extraction and alerting improvements outlined above, the platform would shift from a passive log viewer to an active threat detection system capable of notifying the SOC team without human eyes on the dashboard at all times.

I am happy to implement these changes and present a revised dashboard in a follow-up review.

---

*Moustafa Mohamed — Security Analyst*
*Network Security Lab, March 2026*
