# Part 2 — Snort Intrusion Detection System

## Objective
Deploy and configure Snort as an IDS on the pfSense firewall's LAN and DMZ interfaces, validate detection with controlled ICMP traffic, and enable the GPLv2 Community ruleset.

## Environment
- **IDS:** Snort (pfSense package)
- **Pattern Matching Engine:** AC-BNFA
- **Interfaces monitored:** LAN (em0), DMZ (em2)
- **Blocking Mode:** DISABLED (detection only — IDS mode, not IPS)

---

## Configuration Steps

### 1. Pass List Setup
Created a Pass List to exclude trusted internal traffic from triggering alerts:
- **List Name:** `passlist_LAN_IDS`
- **Assigned Alias:** `LAN_HOME_NETWORK_IDS`
- **Description:** LAN

This prevents Snort from generating false positives on legitimate internal-to-internal traffic.

### 2. LAN Interface Configuration
- Navigated to **Services > Snort > Snort Interfaces > Add**
- Interface: `LAN (em0)`
- Pattern Match Algorithm: `AC-BNFA`
- Blocking Mode: `DISABLED`
- Description: `LAN`
- Saved and started Snort on the interface

### 3. DMZ Interface Configuration
- Added second Snort interface: `DMZ (em2)`
- Same settings as LAN
- Description: `DMZ`
- Started Snort on DMZ interface

### 4. GPLv2 Community Rules Activation
- Navigated to **Services > Snort > Categories > LAN**
- Enabled: **Snort GPLv2 Community Rules**
- Clicked Save — system displayed live-reload confirmation:
  > *"Snort is live-reloading the new ruleset"*

---

## Detection Validation Test

### Test Method
Used pfSense Diagnostics / Ping to generate ICMP traffic:
- **Hostname:** 172.30.0.2
- **IP Protocol:** IPv4
- **Source Address:** DMZ
- **Ping Count:** 3

### Results
```
PING 172.30.0.2 from 172.31.0.1: 56 data bytes
64 bytes from 172.30.0.2: icmp_seq=0 ttl=128 time=0.321 ms
64 bytes from 172.30.0.2: icmp_seq=1 ttl=128 time=0.744 ms
64 bytes from 172.30.0.2: icmp_seq=2 ttl=128 time=0.452 ms
--- 172.30.0.1 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
```

### Snort Alert Log
6 entries generated in the Active Alert Log:

| Date | Proto | Class | Source IP | Source Port | Dest IP | SID | Description |
|------|-------|-------|-----------|-------------|---------|-----|-------------|
| 2026-03-29 13:53:28 | ICMP | Misc activity | 172.31.0.1 | — | 172.30.0.2 | 2100366 | GPL ICMP PING \*NIX |
| 2026-03-29 13:53:28 | ICMP | Misc activity | 172.31.0.1 | — | 172.30.0.2 | 2100368 | GPL ICMP PING BSDtype |

*Pattern repeated for all 3 ping packets (6 total entries — 2 rule matches per packet)*

✅ **Detection confirmed on both LAN and DMZ interfaces.**

---

## Interface Status Overview

| Interface | Snort Status | Pattern Match | Blocking Mode |
|-----------|-------------|---------------|---------------|
| LAN (em0) | ✅ Running | AC-BNFA | DISABLED |
| DMZ (em2) | ✅ Running | AC-BNFA | DISABLED |

---

## Screenshots
- `screenshots/pass-lists-configured.png` — Pass List page showing passlist_LAN_IDS entry
- `screenshots/snort-active-lan.png` — LAN interface with active Snort status
- `screenshots/snort-active-dmz.png` — DMZ interface with active Snort status (both LAN and DMZ visible)
- `screenshots/ping-results.png` — Successful ping from pfSense Diagnostics
- `screenshots/icmp-alerts-snort.png` — 6 ICMP entries in the Snort Active Alert Log
- `screenshots/gplv2-rules-livereload.png` — GPLv2 rules enabled with live-reload confirmation

---

## Notes
- Blocking mode was intentionally left **DISABLED** to operate in pure IDS mode. Enabling blocking (IPS mode) would require additional testing to prevent legitimate traffic from being dropped.
- The AC-BNFA (Aho-Corasick Banded Nondeterministic Finite Automaton) algorithm offers a good balance of speed and memory efficiency for pattern matching.
- SID 2100366 and 2100368 are part of the GPLv2 Community Rules set and match common ICMP ping signatures from Unix and BSD systems respectively.
