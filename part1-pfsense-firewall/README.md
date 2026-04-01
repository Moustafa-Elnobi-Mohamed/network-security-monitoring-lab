# Part 1 — pfSense Firewall Log Configuration

## Objective
Configure the pfSense Community Edition firewall to generate and retain system and firewall traffic logs, then verify that logging is capturing real network events.

## Environment
- **Firewall:** pfSense CE running in VMware Workstation
- **Access:** Web GUI via `172.30.0.1/status_logs.php`
- **Log capacity:** Last 200 general entries / Last 64 firewall entries

## Steps Performed

### System Log Configuration
1. Navigated to **Status > System Logs > General**
2. Verified that syslog was active and capturing system-level events
3. Confirmed entries for: nginx process, filter reload, kernel signals, OpenVPN tunnel management, gateway alarms, and Snort startup events

### Firewall Log Configuration
1. Navigated to **Status > System Logs > Firewall**
2. Verified firewall log was capturing blocked traffic
3. Observed consistent blocked ICMP traffic pattern from WAN

## Key Log Entries Observed

### System Log Highlights
| Time | Process | Message |
|------|---------|---------|
| 13:43:49 | nginx | SSL connection failed (84: Connection reset by peer) |
| 13:43:48 | check_reload_status | Reloading filter |
| 13:43:45 | sysllogd | exiting on signal 15 |
| 13:43:48 | check_reload_status | Spinning firewall |
| 13:41:15 | php-fpm | Successful login for user 'admin' from 172.30.0.2 |
| 13:39:08 | m.gateway_alarm | Gateway alarm: WAN GW2 (Addr:202.20.1.2 Alarm:0 RTT:4.789ms RTTsd:7.287ms Loss:3%) |

### Firewall Log Highlights
- **Action:** Block (✕)
- **Interface:** WAN
- **Source:** 202.20.1.2 (external)
- **Destination:** 202.20.1.1
- **Protocol:** ICMP and UDP
- **Pattern:** Repeated blocked inbound ICMP — consistent with an external ping sweep or reachability test against the WAN IP

## Screenshots
- `screenshots/system-logs.png` — General system log view
- `screenshots/firewall-logs.png` — Firewall traffic log with blocked ICMP entries

## Notes
A Snort startup error appeared in the system log referencing a missing config file path (`/etc/snort/snort_4819_em0`). This is expected at this stage — Snort had not yet been fully configured. The error resolved after completing Part 2.
